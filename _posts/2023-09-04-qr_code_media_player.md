---
layout: post
title:  "QR Code Media Player"
author: robweber
categories: smarthome coding
tags: php linux
---

We were recently trying to organize some of the clutter in our home and I started in on the video game and Blue Ray/DVD collection. For Blue Ray/DVDs (henceforth referred to as just DVDs) we're in the same boat as a lot of households, we just don't use them. Streaming is a good amount of our media consumption but I take movies we've purchased and rip them to our NAS so we can play them back via [Kodi][kodi-tv]. Since the entire collection is digitized what often happens is we look at the DVD rack, select a movie we want, and then find it via Kodi and play it there.

Inspired by many Home Assistant related NFC "jukebox" type projects I thought it would be fun to combine the physical with the digital. I wanted to use the physical disc to trigger the playback of the digital file. Since Kodi has a [JSON-RPC API][kodi-json] being able to trigger playback based on a tag or code seemed doable.

![QR Code](/images/2023-08/qr_code.png)

<!--more-->

{% include toc %}

## NFC vs QR Code

The classic example for this kind of thing is the [Home Assistant RFID Jukebox](https://www.home-assistant.io/blog/2020/09/15/home-assistant-tags/). It was featured on the Home Assistant blog and has had [numerous recreations](https://www.reddit.com/r/homeassistant/search?q=jukebox&restrict_sr=on). The basic idea is you affix an NFC tag to the back of the physical item and code a scanner to read the tag. When the scanner reads the tag it triggers an automation to play the digital item over your TV or speakers. At first glance this is exactly what I wanted to do, but with a movie. As I began to research this more, however, I came up with a few problems that turned me off from this exact approach.

1. Cost - I have over 100 physical DVDs, this would mean purchasing over 100 NFC tags.
2. Maintenance - Each tag would have to be associated with the physical movie somehow, this meant an external database or mapping file
3. Time - each tag would need to be programmed, setup in the software (ESPHome and/or Home Assistant), and affixed to the DVD case.

I thought there must be a way to make this easier and started to think about QR codes instead. These seemed ideal since most smart devices will read a QR code, which can launch a URL. Any data I needed, such as the movie title, could be embedded in the URL. This would mean I could use a single universal web hook to pull out the title and trigger Kodi to play it. Comparing it with NFC I saw the following advantages QR Codes over NFC tags.

1. Cost - I could print the QR codes on labels. Basically $10 max for the whole project.
2. Maintenance - since the QR code would contain the title embedded in a URL I wouldn't need a mapping file or database.
3. Time - After the programming of the web hook the only time involved would be printing and affixing the QR codes. Plus I can easily print more for new movies down the road without touching other parts of the system.

## QR Code Scanning

I quickly threw together a PHP script to lookup a movie in Kodi based on it's title. If I could get this to work then triggering playback would be one simple additional method call. Using an online QR code generator I created a QR code that pointed to my script, appending the title as a query parameter. Something like: `http://webserver/scan.php?title=Inception`. My script was super ugly but here it is below for posterity. Quick note, to run this I already had a webserver with [PHP and cURL][php-curl] installed.

```php
// create the JSON-RPC call
$json_obj = array('jsonrpc'=>'2.0', 'id'=>'1', 'method'=>"VideoLibrary.GetMovies", 'params'=>array('properties'=>array('title', 'runtime', "file", "resume"),
                   'filter'=>array('operator'=>'is','field'=>'title','value'=>$_GET['title'])));

// print the request
echo json_encode($json_obj);
echo "<br />";

// encode the request
$data_string = json_encode($json_obj);

// curl options
$ch_options = array(
  CURLOPT_VERBOSE => 0,
  CURLOPT_RETURNTRANSFER        => true,     // return web page
  CURLOPT_HEADER                => 0,    // don't return headers
  CURLOPT_HTTPHEADER            => array(
  'Content-Type: application/json',
  'Content-Length: ' . strlen($data_string)),

  CURLOPT_ENCODING            => "",       // handle all encodings
  CURLOPT_USERAGENT            => "curl/7.65.1", // who am i
  CURLOPT_POSTFIELDS            => $data_string,

);

// send the request
$ch = curl_init( "http://kodi/jsonrpc" );
curl_setopt_array( $ch, $ch_options );
$info = curl_getinfo($ch);
$content = curl_exec( $ch );
$error_msg = curl_error($ch);
curl_close($ch);

// print the response
echo $content;
echo "<br />";

```

There is a lot going on in there but the line to really pay attention to is the first one:

```
$json_obj = array('jsonrpc'=>'2.0', 'id'=>'1', 'method'=>"VideoLibrary.GetMovies", 'params'=>array('properties'=>array('title', 'runtime', "file"),
                   'filter'=>array('operator'=>'is','field'=>'title','value'=>$_GET['title'])));
```

This calls the [VideoLibrary.GetMovies](https://kodi.wiki/view/JSON-RPC_API/v13#VideoLibrary.GetMovies) method and passes in some options. The most important of which is the filter expression. This filters on a movie based on the query parameter `title`. The response for this request, if the movie exists, would look like this:

```
{"id":"1","jsonrpc":"2.0","result":{"limits":{"end":1,"start":0,"total":1},"movies":[{"file":"//machine/path/to/film/Inception.2010.mkv","label":"Inception","movieid":1188,"runtime":8888,"title":"Inception"}]}}
```

## Generating QR Codes

Once I knew I could do this one time I only had to replicate it for 100 more DVDs. Creating that many QR codes wasn't going to be done with a free online generator. To solve that I had two problems to overcome:

1. Inputting a list of movies.
2. Generating the QR codes in a way I could easily print them.

The list of movies doesn't seem all that difficult but consider the following problem. If I typed _Back To the Future II_ but the movie is called _Back To the Future Part II_ in Kodi then my QR code would be looking up the wrong title. I needed to make sure the encoded title matched the title in Kodi. Solving this was actually pretty simple. Leveraging the Kodi PHP code from above but removing the filter produces a list of all movies. Apply some PHP magic and I [had a page](https://github.com/robweber/qr-code-movie-scanner/blob/main/web/generator.php) where I could select the titles I wanted from the list. This ensured I had exact 1:1 matches from the Kodi database naming and I didn't have to type anything.

![Movie Selector](/images/2023-08/movie_select.png)

Since I was working in PHP the next step was to find a QR code library to generate the actual images. I figured I could generate them in a table that could be easily printed on some label paper for applying to the DVD cases. I found an awesome drop-in [QR code script on Github][qr-code-lib]. I did have a little trouble with it not encoding URL query parameters correctly but a few small modifications fixed that. The result was a landing page where I could select which movies I wanted to generate codes for and once selected these would be sent to another script where the images would generate and display in a table.

## Triggering Playback

At this point I could generate the QR codes easily and trigger a Kodi API check when the code was scanned. To tie this all together I added the critical playback component as well as some quality of life improvements to the code. First off was encapsulating the Kodi API code into it's [own class](https://github.com/robweber/qr-code-movie-scanner/blob/main/web/kodi.php) so I could call different JSON-RPC methods. Once this was done I added two more additional method calls; one to check for already playing content and another to actually play the selected file. At this point I figured "job well done" and went to grab a stack of DVDs to generate codes.

### Revenge of Movie Remakes

As I started to generate the first round of QR codes for printing I came upon an old Disney Classic, _Beauty and the Beast_. This happened to be the 1991 animated edition but immediately behind it was the 2017 live action remake. This was going to be a problem.

My current QR system was built entirely on encoding the movie title and using that to lookup the digital file in Kodi. For movies like _Beauty and the Beast_ that have several versions with the same title this wasn't going to work. Kodi will pull in all matching titles and just play the first one, which may or may not be the one scanned. I realized we had quite a few of these.

The fix was to add yet another variable to the QR encoding scheme - the movie year. This is another attribute Kodi had available but I had to back track and add the year to the generator code and then an additional filter to the scanning code to grab the correct movie title and the correct year. I bit irritating but that's what happens when you're building a project as you go rather than defining a good set of requirements. Lesson learned.

My entire QR code script looks like the following, note the additional `year` filter on the `VideoLibrary.GetMovies` method call.

```
<?php
  include('config.php');
  require('kodi.php');
?>
<html>
  <head>
    <title>Scanner</title>
  </head>
  <body align="center">
    <?php if(hash('sha256', $SECURITY_CODE . $_GET['title']) == $_GET['security']): ?>
    <?php
      $kodi = new KodiComm($DEBUG_MODE);

      $result = $kodi->callMethod('VideoLibrary.GetMovies', array('properties'=>array('title', 'runtime', "file", "resume"),
                                  'filter'=>array('and'=>array(array('operator'=>'is','field'=>'title','value'=>$_GET['title']),
                                                               array('operator'=>'is','field'=>'year','value'=>$_GET['year'])))));

      if($result['result']['limits']['total'] == 1)
      {

        if(!$kodi->isPlaying() || $OVERRIDE_PLAYING)
        {
          if(!$DEBUG_MODE)
          {
            $kodi->callMethod('Player.Open', array('item'=>array('file'=>$result['result']['movies'][0]['file'])));
          }

        ?>
        <h2>Playing <?= htmlspecialchars($_GET["title"]) ?> (<?= $_GET['year'] ?>)</h2>
        <?php
        }
        else
        {
        ?>
        <h2>Cannot play <?= htmlspecialchars($_GET["title"]) ?> (<?= $_GET['year'] ?>), playback in progress</h2>
        <?php
        }
      }
      else
      {
    ?>
    <h2><?= htmlspecialchars($_GET["title"]) ?> (<?= $_GET['year'] ?>) cannot be found</h2>
    <?php
      }
    ?>
  <?php endif ?>
  </body>
</html>
```

I'll admit it's not pretty but I didn't want a lot of overhead so went with straight PHP and HTML, no frameworks. After checking that the passed in `title` and `year` exists some further checks are done. If Kodi is already playing something the process is abandoned - don't want to annoy people. If everything checks out then the found video file is played. A configuration file `config.php` is brought in at the top to store some high level variables. This includes overrides for the playback behavior as well as a debug flag. This is also where the Kodi IP information and base URL for the QR codes is stored.

## Final Steps

Finally everything was ready to generate the QR code labels. With a little help from my family we organized the physical DVDs, generated the QR codes, printed them, and slapped them on the cases. As opposed to buying all the NFC tags this only ended up costing about $10 in label paper. It also has some longevity since I can easily generate new codes for purchased movies down the line. My family was happy that this solution didn't require yet another custom piece of hardware either since everyone has a phone or tablet at the ready most of the time. Guests can even use it when on our Wifi.

For those following along I've posted [all the code for this on Github][qr-code-lib]. It does require a working webserver with PHP and some additional libraries installed. All in all this project went from conception to working model in about 24 hours. That's including the time it took Amazon to deliver the label paper. This was a pretty fun weekend exercise and it adds a fun element to picking out a movie that my family enjoys.

## Links

* [QR Code Movie Scanner][qr-code-lib] - full code
* [Kodi Media Center][kodi-tv] - Media center software we use for digital file playback
* [Kodi JSON-RPC API][kodi-json] - documentation for the JSON-RPC service
* [PHP cURL][php-curl] - documentation for cURL use with PHP

[kodi-tv]: https://kodi.tv
[kodi-json]: https://kodi.wiki/view/JSON-RPC_API/v13
[php-curl]: https://www.php.net/manual/en/book.curl.php
[qr-code-lib]: https://github.com/psyon/php-qrcode
