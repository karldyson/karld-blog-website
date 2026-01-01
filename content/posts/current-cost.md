+++
title = "CurrentCost Electricity Monitoring"
date = 2012-09-16
[extra]
toc = false
toc_inline = true
toc_ordered = true
styles = ["posts/demo.css"]
[taxonomies]
tags = ["linux","electricity","currentcost","monitoring","raspberry-pi","perl","munin"]
+++

For a while now, I've been using a Current Cost CC128 to monitor my electricity usage.

I hacked together a munin plugin to monitor both the electricty usage, and as the unit is in my lounge and coughs up temperature, it also provides the temperature reading.

For a number of reasons, the munin plugin doesn't read the device directly. Only one device can open the serial port at a time, and there were a couple of things I wanted to do with the data, so didn't want them fighting. I had a small piece of code grab the XML once a minute and dump it in a file instead.

Talking to a friend recently, they quite rightly pointed out (reminding me) that this method takes an instantaneous value for both the temperature and the electricity usage.

Whilst not a problem for the former as it won't vary greatly within the 5 minute polling periods, the electricity usage clearly will. Think about boiling a kettle; if you were (un)? lucky enough to time the kettle just right, you could be between polls and not record the increased usage at all.

So, I've modified the way in which I record the data and write the files for the various things including munin to use.

As you'll probably be aware, the CC128 outputs it's data once every 6 seconds, so there's clearly more data to make use of than just the single reading every 5 minutes.

I have a small daemon written in perl that listens on the serial port and every time it gets a full string of XML, it records the data to a database. Rather than re-write the munin plugin at the moment, along with anything else that uses the data file, I have a small bit of perl that runs from cron each minute that populates the expected XML into the file using averages of the values collected in the preceding 5 minute period. You could, of course, make this 95th percentile or whatever tickles your fancy.

So, the SQL table looks like this:

```sql
mysql> describe data;
+------------+------------------+------+-----+-------------------+-----------------------------+
| Field      | Type             | Null | Key | Default           | Extra                       |
+------------+------------------+------+-----+-------------------+-----------------------------+
| id         | int(10) unsigned | NO   | PRI | NULL              | auto_increment              |
| dsb        | int(10) unsigned | YES  |     | NULL              |                             |
| time       | timestamp        | NO   |     | CURRENT_TIMESTAMP | on update CURRENT_TIMESTAMP |
| tmpr       | decimal(6,4)     | YES  |     | NULL              |                             |
| sensor     | int(10) unsigned | YES  |     | NULL              |                             |
| unit_id    | int(10) unsigned | YES  |     | NULL              |                             |
| type       | int(10) unsigned | YES  |     | NULL              |                             |
| channel    | int(10) unsigned | YES  |     | NULL              |                             |
| data_units | varchar(128)     | YES  |     | NULL              |                             |
| data_value | int(10) unsigned | YES  |     | NULL              |                             |
+------------+------------------+------+-----+-------------------+-----------------------------+
```

```sql
create or replace view averages as
select sensor,dsb,unit_id,type,channel,data_units,avg(tmpr) as tmpr,avg(data_value) as value,now() as now,
from_unixtime(300 * floor(unix_timestamp()/300)-300) as start,from_unixtime(300 * floor(unix_timestamp()/300)) as end 
from data 
where time between from_unixtime(300 * floor(unix_timestamp()/300)-300) and from_unixtime(300 * floor(unix_timestamp()/300)) 
group by unit_id,sensor,channel;
```

I should credit [http://larig.wordpress.com/2012/02/08/time-rounded-to-five-minutes-in-mysql/](http://larig.wordpress.com/2012/02/08/time-rounded-to-five-minutes-in-mysql/) for the sql to round time to the current 5 minute window.

The bit that writes the XML file looks like this:

```perl
#!/usr/bin/perl

use strict;

use DBI;

my $dsn = "DBI:mysql:database=<removed>;host=<removed>;port=3306";
my $dbh = DBI->connect($dsn, '<removed>', '<removed>');

my $sql = 'select * from averages';

my $sth = $dbh->prepare($sql) || die "cannot prepare sql\n$sql\n$DBI::errstr\n";

$sth->execute() || die "cannot execute sql\n$sql\n$DBI::errstr\n";

if($sth->rows > 0) {
	my $r;
	print "<msg>\n";
	while(my $row = $sth->fetchrow_hashref) {
		print "  <ch$row->{channel}>\n";
		print "    <$row->{data_units}>$row->{value}</$row->{data_units}>\n";
		print "  </ch$row->{channel}>\n";
		$r = $row;
	}
	print "  <sensor>$r->{sensor}</sensor>\n";
	print "  <id>$r->{unit_id}</id>\n";
	print "  <dsb>$r->{dsb}</dsb>\n";
	print "  <tmpr>$r->{tmpr}</tmpr>\n";
	print "  <type>$r->{type}</type>\n";
	print "  <time>$r->{now}</time>\n";
	print "</msg>\n";
} else {
	die "no rows returned!\n";
}

$sth->finish;
$dbh->disconnect;
```

...and lastly, the bit that grabs the stuff and chucks it into the database looks like this (and also writes a backward compatible version of the XML file with the instantaneous values...:

```perl
#!/usr/bin/perl

$|++;

use strict;

use File::Copy qw/move/;
use Device::SerialPort qw/:PARAM :STAT 0.07/;
use XML::Simple;
use DBI;
use Data::Dumper;

my $debug = $ENV{DEBUG} || 0;

use Proc::PID::File;
if(Proc::PID::File->running()) {
	print "Already running\n" if $debug;
	exit;
}

my $port = $ENV{port} || '/dev/ttyUSB0';
my $baud = $ENV{baud} || 57600;

my $debug = $ENV{DEBUG} || 0;

my $cc = Device::SerialPort->new($port) || die "unable to open port: $!\n";
$cc->baudrate($baud);
$cc->read_char_time(0);
$cc->read_const_time(1000);
$cc->write_settings;

my $dsn = "DBI:mysql:database=<removed>;host=<removed>;port=3306";
my $dbh = DBI->connect($dsn, '<removed>', '<removed>');
my $sql = 'insert into data (dsb,tmpr,sensor,unit_id,type,channel,data_units,data_value) values (?,?,?,?,?,?,?,?)';
my $sth = $dbh->prepare($sql) || die "cannot prepare sql\n$sql\n$DBI::errstr\n";

while(1) {
	my $timeout = 10;
	my $chars = 0;
	my $buffer;
	LOOP: while($timeout > 0) {
		my($count, $saw) = $cc->read(255);
		if($count > 0) {
			$chars += $count;
			$buffer .= $saw;
			chomp $buffer;
			print "added\n$saw\nnow have\n$buffer\n" if $debug;
			if($buffer =~ m|<msg>.*?</msg>|) {
				print "Buffer contains XML\n\n" if $debug;
				do_xml($buffer);
				last LOOP;
			}
			elsif($buffer =~ m|</msg>|) {
				print "Caught the end of the XML\n" if $debug;
				undef $buffer;
			}
		} else {
			print "." if $debug;
			$timeout--;
		}
	
		print "Waited $timeout and never saw what I was looking for\n" if $timeout == 0 && $debug;
	
	}
}

sub do_xml {
	my $xmls = shift;
	if($xmls =~ "<hist>.*?</hist>") {
		return;
	} else {
		$XML::Simple::PREFERRED_PARSER = 'XML::Parser';
		my $xml;
		eval { $xml = XMLin($xmls); };
		return undef if $@;
		print Dumper($xml) if $debug;
		for(keys %$xml) {
			if(m/^ch(\d+)$/) {
				my $channel = $1;
				my $name = 'ch'.$channel;
				for(keys %{$xml->{$name}}) {
					my $unit = $_;
					my $value = $xml->{$name}->{$unit};
					printf("dsb[%s] tmpr[%s] sensor[%s] id[%s] type[%s] ch[%s] units[%s] val[%s]\n",$xml->{dsb},$xml->{tmpr},$xml->{sensor},$xml->{id},$xml->{type},$channel,$unit,$value) if $debug;
					$sth->execute($xml->{dsb},$xml->{tmpr},$xml->{sensor},$xml->{id},$xml->{type},$channel,$unit,$value) || die "cannot execute sql\n$sql\n$DBI::errstr\n";
				}
			}
		}
		open(XML,'>/tmp/currentcost.snapshot.xml.tmp') || die "cannot open file: $!\n";
		print XML $xmls || die "cannot write to file: $!\n";
		close XML || die "cannot close file: $!\n";
		move('/tmp/currentcost.snapshot.xml.tmp', '/tmp/currentcost.snapshot.xml') || die "unable to move file: $!\n";
	}
}
```

I restart that from cron periodically, incase it dies. If it's already running, it simply exits silently. It's a quick hacky way to make it restart in the event something horrible happens. I did at least put the XML parse in an `eval` in case it barfs :-)

As usual, this blog post is as much for my personal notes of how I did it than anything else. Some of the code is awful, but it's not life or death and so I've often been lazy with it. There's no doubt this could be done prettier and less hacky if I had more time :-)
