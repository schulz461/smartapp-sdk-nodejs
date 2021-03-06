# SmartThings SmartApp NodeJS SDK (preview)

<p align="center">
<a href="https://www.npmjs.com/package/@smartthings/smartapp"><img src="https://badgen.net/npm/v/@smartthings/smartapp" /></a>
<a href="https://www.npmjs.com/package/@smartthings/smartapp"><img src="https://badgen.net/npm/license/@smartthings/smartapp" /></a>
<a href="https://status.badgen.net/"><img src="https://badgen.net/xo/status/@smartthings/smartapp" /></a>
<a href="https://lgtm.com/projects/g/SmartThingsCommunity/smartapp-sdk-nodejs/context:javascript"><img alt="Language grade: JavaScript" src="https://img.shields.io/lgtm/grade/javascript/g/SmartThingsCommunity/smartapp-sdk-nodejs.svg?logo=lgtm&logoWidth=18"/></a>
<a href="https://lgtm.com/projects/g/SmartThingsCommunity/smartapp-sdk-nodejs/alerts/"><img alt="Total alerts" src="https://img.shields.io/lgtm/alerts/g/SmartThingsCommunity/smartapp-sdk-nodejs.svg?logo=lgtm&logoWidth=18"/></a>
<a href="https://snyk.io/test/github/SmartThingsCommunity/smartapp-sdk-nodejs?targetFile=package.json"><img src="https://snyk.io/test/github/SmartThingsCommunity/smartapp-sdk-nodejs/badge.svg?targetFile=package.json" alt="Known Vulnerabilities" data-canonical-src="https://snyk.io/test/github/SmartThingsCommunity/smartapp-sdk-nodejs?targetFile=package.json" style="max-width:100%;"></a>
<a href="https://smartthingsdev.slack.com/messages/CG595N08N"><img src="https://badgen.net/badge//smartthingsdev?icon=slack" /></a>
</p>

[Reference Documentation](doc/index.md)

SDK that wraps the SmartThings REST API and reduces the amount of code necessary to write a SmartApp app. It supports both webhook and AWS Lambda implementations. This is a preview version of the API and will change over time time.

## Installation

```bash
npm i @smartthings/smartapp --save
```

## Importing

`NodeJS`:

```javascript
const smartapp = require('@smartthings/smartapp')
```

Or, if you're transpiling to `ES6`/`ES2015`+:

```javascript
import smartapp from '@smartthings/smartapp'
```

## Highlights

- [x] Javascript API hides details of REST calls and authentication.
- [x] Event handler framework dispatches lifecycle evebnts to named event handlers.
- [x] Configuration page API simplifies page definition.
- [x] Integrated [i18n](https://www.npmjs.com/package/i18n) framework provides configuration page localization.
- [x] [Winston](https://www.npmjs.com/package/winston) framework manges log messages.

## Example

The following example is the equivalent of the original SmartThings Groovy _Let There Be Light_ app that turns on and off a light when a door opens and closes.

### Running it as a web service

To run the app with an HTTP server, like Express.js:

```javascript
const express    = require('express');
const smartapp   = require('@smartthings/smartapp');
const server     = module.exports = express();
const PORT       = 8080;

server.use(express.json());

// @smartthings_rsa.pub is your on-disk public key
// If you do not have it yet, omit publicKey()
smartapp
    .publicKey('@smartthings_rsa.pub') // optional until app verified
    .configureI18n()
    .page('mainPage', (context, page, configData) => {
        page.section('sensors', section => {
           section.deviceSetting('contactSensor').capabilities(['contactSensor']).required(false);
        });
        page.section('lights', section => {
            section.deviceSetting('lights').capabilities(['switch']).multiple(true).permissions('rx');
        });
    })
    .installed((context, installData) => {
        console.log('installed', JSON.stringify(installData));
    })
    .uninstalled((context, uninstallData) => {
        console.log('uninstalled', JSON.stringify(uninstallData));
    })
    .updated((context, updateData) => {
        console.log('updated', JSON.stringify(updateData));
        context.api.subscriptions.unsubscribeAll().then(() => {
              console.log('unsubscribeAll() executed');
              context.api.subscriptions.subscribeToDevices(context.config.contactSensor, 'contactSensor', 'contact', 'myDeviceEventHandler');
        });
    })
    .subscribedEventHandler('myDeviceEventHandler', (context, deviceEvent) => {
        const value = deviceEvent.value === 'open' ? 'on' : 'off';
        context.api.devices.sendCommands(context.config.lights, 'switch', value);
        console.log(`sendCommands(${JSON.stringify(context.config.lights)}, 'switch', '${value}')`);

        /* All subscription event handler types:
         *   - DEVICE_EVENT (context, deviceEvent)
         *   - TIMER_EVENT (context, timerEvent)
         *   - DEVICE_COMMANDS_EVENT (context, deviceId, command, deviceCommandsEvent)
         *   - MODE_EVENT (context, modeEvent)
         *   - SECURITY_ARM_STATE_EVENT (context, securityArmStateEvent)
        */
    });

/* Handle POST requests */
server.post('/', function(req, res, next) {
  console.log(`${new Date().toISOString()} ${req.method} ${req.path} ${req.body && req.body.lifecycle}`);
  smartapp.handleHttpCallback(req, res);
});

/* Start listening at your defined PORT */
server.listen(PORT, () => console.log(`Server is up and running on port ${PORT}`));
```

### Running as an AWS Lambda function

To run as a Lambda function instead of an HTTP server, ensure that your main entry file exports `smartapp.handleLambdaCallback(...)`.

> **Note:** This snippet is heavily truncated for brevity – see the web service example above a more detailed example of how to define a `smartapp`.

```javascript
const smartapp = require('@smartthings/smartapp')
smartapp
    .page( ... )
    .updated(() => { ... })
    .subscribedEventHandler( ... );
exports.handle = (event, context, callback) => {
    smartapp.handleLambdaCallback(event, context, callback);
};
```

### Localization

Configuration page strings are specified in a separate `locales/en.json` file, which can be automatically created the first time you run the app. Here's a completed English localization file for the previous example:

```json
{
  "pages.mainPage.name": "Let There Be Light",
  "pages.mainPage.sections.sensors.name": "When this door or window opens or closes",
  "pages.mainPage.settings.contactSensor.name": "Select open/close sensor",
  "pages.mainPage.sections.lights.name": "Turn on and off these lights and switches",
  "pages.mainPage.settings.lights.name": "Select lights and switches",
  "Tap to set": "Tap to set"
}
```

---

## More about SmartThings

If you are not familiar with SmartThings, we have
[extensive on-line documentation](https://smartthings.developer.samsung.com/develop/index.html).

To create and manage your services and devices on SmartThings, create an account in the
[developer workspace](https://devworkspace.developer.samsung.com/).

The [SmartThings Community](https://community.smartthings.com/c/developers/) is a good place share and
ask questions.

There is also a [SmartThings reddit community](https://www.reddit.com/r/SmartThings/) where you
can read and share information.

## License and Copyright

Licensed under the [Apache License, Version 2.0](LICENSE)

Copyright 2019 SmartThings, Inc.