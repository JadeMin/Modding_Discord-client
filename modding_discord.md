# Modding the discord client
Disclaimer: Modding the discord client is against Discord's term of service. I'm not responsible if any action is taken against you.
## Why modding ?
Modding the client will allow you to customize to: 
- change the appearance of the discord client
- have script to automaticaly send messages at specific times
- log messages from a chat into a local file

This guide will explain how to inject js code into the discord client and send messages.
I'm assuming you know you to write javascript code and are using Windows.
Most of this will also work on Linux and Mac but the directories will not be the same

## How to mod discord ?

First install `npm` and `asar`

npm can be installed from here: https://nodejs.org/en/ (it's bundled with nodejs)

asar can when be installed with `npm install asar -g`

Then go to the following directory: `%LocalAppData%/Discord/app-<some version>/modules/discord_desktop_core-<number>/discord_desktop_core/`
You will find a file named `core.asar`

asar files work find of like zip files. You should make a backup of the file as we are going to modify it.

To unpack the file, run `asar extract core.asar unpacked`
This will create an `unpacked` folder. Inside, search for `app/mainScreen.js`.
This is the js file we will edit to inject js code into Discord.

I suggest adding the following lines (put you could add anything else depending on the feature you want)
At line ~350, inside the object `mainWindowOptions.webPreferences` add the property: `nodeIntegration: true`.
This will allow you to access node js function from inside discord to write files for example.

At line ~440, in the scope of launchMainAppWindow, add:
```js
const fs = require('fs');
mainWindow.webContents.on('dom-ready', () => {
  setTimeout(() => {
    mainWindow.webContents.executeJavaScript(fs.readFileSync('C:/discord.js') + "");
  }, 3000);
});
```
This will load the file located at `C:/discord.js` into the discord client.
You can change this path to any path convenient to you.

You will then need to repack this code into the asar file. To do this, run:

`asar pack unpacked core.asar`

This will override the `core.asar` file so making a backup is important here.

When, create the js file at `C:/discord.js` and add the following inside:
(You can customize this to do other stuff)
```js
// Discord requires a authorization token to send messages.
// We will fetch this token by listening to request discord makes
let authTokenObtained = false;
let insideChannelRequest = false;
let authToken = "";
let xsuperToken = "";

// We override the default AJAX request to listen to the requests discord makes.
var proxied1 = window.XMLHttpRequest.prototype.open;
window.XMLHttpRequest.prototype.open = function() {
	console.log(arguments[1]);
	if(arguments[1].startsWith('https://discordapp.com/api/v6/channels')){
		if(!authTokenObtained){
			insideChannelRequest = true;
		}
		(function(url){
			setTimeout(function(){ // wait for the ui to update.
				let elements = document.getElementsByTagName('h3');
       				// We update the UI to display the channel name.
				if(elements.length >= 1 && elements[0].style.userSelect !== "text"){ // add channel URL to name
					elements[0].style.userSelect = "text";
					elements[0].innerHTML += " | "+url.substring("https://discordapp.com/api/v6/channels".length);
				}
			},1000);
		})(arguments[1]);
	}
	return proxied1.apply(this, [].slice.call(arguments));
};

let proxied2 = window.XMLHttpRequest.prototype.setRequestHeader;
window.XMLHttpRequest.prototype.setRequestHeader = function() {
	if(insideChannelRequest && !authTokenObtained && arguments[0] === "Authorization"){
		authToken = arguments[1];
		authTokenObtained = true;
		console.log("Token Obtained.");
	}
	if(insideChannelRequest && arguments[0] == "X-Super-Properties"){
		xsuperToken = arguments[1];
		console.log("X-Super-Properties updated.")
	}
	// console.log( arguments );
	return proxied2.apply(this, [].slice.call(arguments));
};

// We can now define a sendMessage function that behaves exactly like discord:
function sendMessage(msg,channel){
	if(!authTokenObtained){
		console.log("Unable to send message without authToken.");
		console.log("Try typing something in a chat to obtain the token.");
	}
	channel_url = `https://discordapp.com/api/v6/channels/${channel}/messages`;

	request = new XMLHttpRequest();
	request.withCredentials = true;
	request.open("POST", channel_url);
	request.setRequestHeader("Content-Type", "application/json");
	request.setRequestHeader("Authorization", authToken);
	request.setRequestHeader("X-Super-Properties", xsuperToken);
	request.send(JSON.stringify({ content: msg, tts: false }));
}

```

Now, once you reload discord, you can see the channel id name of somebody next to their username.
Also, when opening the console (with Ctrl+Shift+I) inside discord, you can run:
`sendMessage("<your message here>","<the channel id here>");` which will send a message in the corresponding channel.

You can use the same stuff to listen to incomming messages (you override the AJAX request callbacks to get access to the messages fetched by discord)
You can also edit the `discord.js` file to change the css. The code above already does that to make usernames selectable.

Now, you can write in the console something like for example:
```js
setTimeout(function(){
  sendMessage("I'm still online, I am not afk","<the channel id>");
},1000 * 60 * 60);
```
to send a message in one hour. You can get creative and do all sort of stuff with this.

This concludes this guide. Have fun.
