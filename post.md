## 1 Introduction

Twilio's MMS messaging has been available for a year, but it was only useable in the U.S. via short codes. Canadian users could access it via both short and long codes.

Finally, Twilio has made MMS messaging available to U.S. via long codes as well. In short, this means that you can now use any Twilio phone number to send and receive picture messages. 

This can be used for various purposes, from help desks to live blogging, sending concert tickets, accepting photos to accompany an insurance claim, or just sharing that incredible animated GIF you saw on Tumblr with your friends.

You can also combine MMS messaging with two-factor authentication to create a unique type of CAPTCHA experience to keep unwanted users out of your website.

In this article, we are going to build a system using Twilio MMS messaging and PHP that will ask users to enter their phone number, then generate a unique CAPTCHA image and send it to their mobile phone. Users will then have to type in that code to get entrance to our system &mdash; this way, we know they are an actual user.

This could later be integrated into a database that already contains their phone numbers.

## 2 Getting set up

Before we can start sending and receiving MMS messages, we have a little bit of work to do. We need to sign up for a Twilio account, for starters, and then obtain an MMS-enabled phone number to use with our application. Let’s get that taken care of so we can move on to the code.

If you don’t already have one, [sign up for a Twilio account now](http://www.twilio.com/try-twilio). During the signup process, you’ll be prompted to search for and buy a number. You can go ahead and just take the default number for now.

<img src="https://dl.dropboxusercontent.com/u/461614/twilio/Twilio_-_Try_Twilio_Free-20140916-210209.png" />

After you secure your number, you might want to play around with it and send yourself a few text messages, or maybe receive a phone call. Once you’re ready to get to the MMS action, click the button to go to your account.

<img src="https://dl.dropboxusercontent.com/u/461614/twilio/Twilio_Signup_-_Success-20140917-184126.png" />

### 2.1 Buy an MMS-enabled phone number

On your [account dashboard](https://www.twilio.com/user/account), you’ll see your account SID and auth token near the top of the screen. These are like your username and password for the Twilio API – you’ll need these values later on to make authenticated requests.

Right now, we need to either buy a new number or use an existing one that can send MMS messages. Click on “Numbers” in the top nav &mdash; in the list of your phone numbers, icons indicate the capabilities of each number. If you already have a number with MMS enabled, then you’re all set!

If you still need a number with MMS, click the “Buy Number” button. Use the form to search for a number that has MMS capabilities by ticking the "MMS" checkbox.

<img src="https://dl.dropboxusercontent.com/u/461614/twilio/Twilio_User_-_Account_Phone_Numbers_Search-20140916-211349.png">

Now choose an MMS-enabled number from the results list. Now that we have a number capable of sending and receiving MMS messages, we’re ready to write code. We’ll start sending our first pictures by using [a high level helper library](https://www.twilio.com/docs/libraries) that makes it easier to work with the Twilio API.

In this case, we'll use the PHP helper library, as we will be building this application in PHP.

### 2.2 Set up our project

We are going to use a handy PHP utility called Composer to set up the libraries that we will be using.

If you have not yet installed Composer, then please do:

<!--code lang=bash linenums=true-->

    $ curl -s https://getcomposer.org/installer | php

Then, create the following file called `composer.json`:

<!--code lang=json linenums=true-->

	{
		"require": {
		        "twilio/sdk": "dev-master",
		        "gregwar/captcha": "dev-master",
		        "gregwar/image": "dev-adapter"
		}
	}

Now, from your terminal, run:

<!--code lang=bash linenums=true-->

    $ php composer.phar install

This will create your `vendor/` folder with the required libraries; you will also want to create a folder named `output` to temporarily store the captcha images.

Our system will consist of three files. First, let's create `index.php`:

<!--code lang=php linenums=true-->
```php
<?php
session_start();
require 'vendor/autoload.php';
use Gregwar\Captcha\CaptchaBuilder;
?>
<!doctype html>
<html lang="en">
<head>
    <style>
    input[type="text"] {
        width: 200px;
        display: block;
        margin: 10px 0;
    } 
    </style>
</head> 
<body>
    <h1>Halt! Who Goes There?</h1>
    <p>Before you can go any further, you have to first enter your phone number so we can make sure you are a human.</p>
    <form action="send.php" method="POST">
        <input type="text" name="phone" placeholder="enter a phone number"/>
        <input type="submit" value="I'm a human!!!!"/>
    </form>
</body>
</html>
```

Now, create `send.php`:

<!--code lang=php linenums=true-->
```php
<?php
session_start();
require 'vendor/autoload.php';
use Gregwar\Captcha\CaptchaBuilder;

$twilio_account_sid = 'YOUR-TWILIO-ACCOUNT-SID';
$twilio_auth_token = 'YOUR-TWILIO-AUTH-TOKEN';
$twilio_from_number = 'YOUR-TWILIO-PHONE-NUMBER';

$builder = new CaptchaBuilder;
$builder->build();

$_SESSION['phrase'] = $builder->getPhrase();

$file = 'output/'.uniqid().'.jpg';		
$builder->save( $file );

$_SESSION['file'] = $file;

$media_url = 'http://'.$_SERVER['HTTP_HOST'].'/'.$file;

$client = new Services_Twilio($twilio_account_sid, $twilio_auth_token);
$to_number = $_POST['phone'];
$message = $client->account->messages->sendMessage(
	$twilio_from_number, 
	$to_number,
	"Please type this code in on the site", 
	$media_url
);
echo '<!-- Message sent! SID: ' . $message->sid.' -->';
?>
<!doctype html>
<html lang="en">
<head>
    <style>
    input[type="text"] {
        width: 200px;
        display: block;
        margin: 10px 0;
    } 
    </style>
</head> 
<body>
    <h1>Verify Yourself!</h1>
    <p>What is the code we just sent to your mobile phone?</p>
    <form action="verify.php" method="POST">
        <input type="text" name="code" placeholder="enter your code"/>
        <input type="submit" value="Verify me please"/>
    </form>
</body>
</html>
```

Our final file will be `verify.php`:

<!--code lang=php linenums=true-->
```php
<?php
session_start();
require 'vendor/autoload.php';
use Gregwar\Captcha\CaptchaBuilder;

$twilio_account_sid = 'YOUR-TWILIO-ACCOUNT-SID';
$twilio_auth_token = 'YOUR-TWILIO-AUTH-TOKEN';
$twilio_from_number = 'YOUR-TWILIO-PHONE-NUMBER';

unlink( $_SESSION['file'] );
if( $_POST['code'] != $_SESSION['phrase'] ){
	header("location: index.php");
	exit;
}
?>
<!doctype html>
<html lang="en">
<head>
</head> 
<body>
    <h1>Super Secret Area</h1>
    <p>You have just gained access to the super secret area</p>
    <p>The secret answer to everything is <strong>42</strong>.</p>
    <p><em>But don't tell anybody that</em></p>
</body>
</html>
<?php
```

Let's break down what we've done here:

- \* `index.php` will prompt the visitor to enter their phone number.
- \* `send.php` will generate the unique image and send it to their mobile phone. It will then display a form for the user to enter the code we sent them.
- \* `verify.php` will then check the code they entered against the code they were sent, and either return them to `index.php` or display a message.

This is a rudimentary example of using Twilio MMS messaging, but it should give you a good base to see how you can handle it.

### 2.3 Run it locally

Let's test our system locally. For local development, PHP can act as a web server, so let's set up a file to tell PHP how to act. 

Create `server.php`:

<!--code lang=php linenums=true-->
```php
<?php
//	php -S 127.0.0.1:8888 server.php
if (preg_match('/\.(?:png|jpg|jpeg|gif|css|js|php)$/', $_SERVER["REQUEST_URI"])) {
    return false;
} else {
    include __DIR__ . '/index.php';
}
```

Now, from the terminal, we can run `$ php -s 127.0.0.1:8888 server.php`. If we visit [http://127.0.0.1:8888](http://127.0.0.1:8888) in our browser, we should be able to access our site.

## 3 Get a public URL for your app with ngrok

Now that we have an endpoint which can respond to an incoming request from Twilio, we need to get it on the Internet. We could deploy it to a web host, but it's usually nice to have Twilio hit the currently-running local version of your application during development. So how can we put our laptop on the Internet?

Enter one of our very favorite tools, [ngrok](http://www.ngrok.com/). ngrok forwards HTTP traffic from a public subdomain to a local port on your computer. To use ngrok, you simply [download the binary executable](https://ngrok.com/download) for your system, navigate to the directory where you downloaded it in a Terminal window, and execute the command, passing in the port number you’re running your app on locally.

Using ngrok can be a bit more involved if you’re running on Windows. [Check out this tutorial for in-depth information on getting ngrok going on that OS](https://www.twilio.com/blog/2014/03/configure-windows-for-local-webhook-testing-using-ngrok.html). For more guidance on using and configuring ngrok generally, [you might check out this post as well](https://www.twilio.com/blog/2013/10/test-your-webhooks-locally-with-ngrok.html).

If you’ve been following the instructions so far, your application should be running on port 8888. In your Mac or \*nix terminal, type `$ ./ngrok 8888`, or just `ngrok 8888` on Windows. The result should be an interface like this one that contains your local port’s new public URL.

<img src="https://dl.dropboxusercontent.com/u/461614/twilio/ngrok_%E2%80%94_80%C3%9724-20140917-123611.png" />

See that `http://` URL in the terminal output? Copy it to your clipboard; this is the URL to your website.

**Why use ngrok?** Because in order to send images using Twilio MMS messaging, the image needs a public URL. ngrok gives you this if you are working locally; if you are hosting these files on a web server somewhere, then you can ignore the ngrok instructions.

## 4 Wrapping Up

This post has introduced how to send MMS messages with Twilio. You can use these basic building blocks and create entirely new categories of messaging applications. 

If you have any questions, don't hesitate to book me for an AirPair!