+++
title = "Live Electricity Usage"
description = "My set up to have live monitoring and graphing of my electricity usage"
date = 2012-10-02
[extra]
toc = false
[taxonomies]
tags = ["linux","electricity","currentcost","monitoring","raspberry-pi","perl","munin"]
+++

<figure>
{{ image(url="live-electricity-graph.png") }}
<figcaption>Live Electricity Usage Graph</figcaption>
</figure>

So, following on from my [recent post on updating my Currentcost code](@/posts/current-cost.md) I now have every update coming from the Currentcost unit popping directly into the database, every 6 seconds.

I had become aware of the [Highcharts charts](http://highcharts.com/) javascript and when reading through their [Live Charts Demo](http://www.highcharts.com/documentation/how-to-use#live-charts) decided that this was just waiting to be played with.

I now have essentially the same graph that they have in that demo, and the page HTML is very similar. The difference is obviously to call my code instead, and also a change to refresh every 6 seconds:

```javascript
function requestData() {
    $.ajax({
        url: '/power/latest.php',
        success: function(point) {
            var series = chart.series[0],
                shift = series.data.length > 60; // shift if the series is longer than 20

            // add the point
            chart.series[0].addPoint(eval(point), true, shift);

            // call it again after one second
            setTimeout(requestData, 6000);
        },
        cache: false
    });
}
```

My AJAX call is also to some PHP, but mine looks up the latest data from the database with a simple SQL statement:

```sql
select unix_timestamp(time) as time,data_value from data order by time desc limit 1
```

I quickly realised that this would slap the database, and more so if more than one person was viewing the page. That would be unnecessary, and so I installed `memcached` and the `php5` `memcached` module.

```php
<?php header("Content-type: text/json"); $m = new Memcached(); $m->addServer('127.0.0.1', 11211);

if(!($row = $m->get('leccy'))) {
        //echo "didn't get from memcache\n";
        $row = get_from_db();
        $m->set('leccy', $row, 6);
}
else if(time() - $row['time'] > 5) {
        //echo "got, but is old\n";
        $row = get_from_db();
        $m->set('leccy', $row, 6);
}

// x is JS time which is unixtime x 1000
$x = $row['time'] * 1000;

// y is the value at that point
$y = $row['data_value'] * 1;

$ret = array($x, $y);

echo json_encode($ret);

function get_from_db() {
        mysql_connect("db server","db user","db passwd") or die("cannot connect: ".mysql_error());
        mysql_select_db("db name");
        $sql = 'select unix_timestamp(time) as time,data_value from data order by time desc limit 1';
        $sth = mysql_query($sql) or die("no query: ".mysql_error());
        $row = mysql_fetch_assoc($sth);
        mysql_close();
        return $row;
}

?>
```

Edit: Slight tweak to the code, as there was some obvious lazyness that I tidied up.

Enjoy.

