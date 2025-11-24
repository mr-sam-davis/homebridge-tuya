# Advanced Options

**During the beta version, the options are unstable, may get changed during updates.**

The main function of `deviceOverrides` is to convert "non-standard schema" to "standard schema", making the device compatible with this plugin.

Before configuring, you may need to:
- Have basic programming skills in JavaScript (Only used in `onGet`/`onSet` handlers).
- Understand the concept of device schema (also known as Data Type): [Tuya IoT Development Platform > Cloud Development > Standard Instruction Set > Data Type](https://developer.tuya.com/en/docs/iot/datatypedescription?id=K9i5ql2jo7j1k)
- Read the documentation of your device product in [SUPPORTED_DEVICES.md](./SUPPORTED_DEVICES.md).
- Obtain device info json from `/path/to/persist/TuyaDeviceList.xxx.json` (the full path can be found from logs).
- Locate any "incorrect schema" in your device info json, and convert it to the "correct schema".


### Configuration

`options.deviceOverrides` is an **optional** array of device overriding config objects, which is used for converting "non-standard schema" to "standard schema", making the device compatible with this plugin. The plugin resolves overrides in three layers:

1. **Device or Scene match**: an entry whose `id` matches the device `id` or `uuid`.
2. **Product match**: an entry whose `id` equals the device `product_id`.
3. **Global defaults**: an entry whose `id` is `global` that applies to every device.

Only one entry per `id` is allowed; duplicated `id` or duplicated schema `code` within an entry will fail validation. Schema lookups use the first matching override, so avoid overlapping definitions across the three layers.

Each element in the array is described as follows:

- `id` - **required**: Device ID, Product ID, Scene ID, or `global`.
- `category` - **optional**: Device category code. See [SUPPORTED_DEVICES.md](./SUPPORTED_DEVICES.md). Also you can use `hidden` to hide the device, product, or scene. **⚠️Overriding this property may lead to unexpected behaviors and exceptions, so please remove the accessory cache after making changes.**
- `unbridged` - **optional**: Unbridge accessories. Defaults to `false`.
- `adaptiveLighting` - **optional**: Adaptive Lighting. Defaults to `false`. Not all light device support this feature, please use it on demand.
- `schema` - **optional**: An array of schema overriding config objects, used for describing datapoint (DP). When your device has non-standard DP, you need to transform them manually with configuration. Each element in the schema array is described as follows:
  - `code` - **required**: DP code in the device payload. This is the value the cloud reports and the plugin sends by default.
  - `newCode` - **optional**: Alias DP code exposed to HomeKit. Use this to rename a DP while keeping the device payload unchanged.
  - `type` - **optional**: New DP type. One of `Boolean`, `Integer`, `Enum`, `String`, `Json`, or `Raw`.
  - `property` - **optional**: New DP property object. For `Integer` type, the object should contain `min`, `max`, `scale`, and `step`. For `Enum` type, the object should contain `range`. For more information, see `TuyaDeviceSchemaProperty` in [TuyaDevice.ts](./src/device/TuyaDevice.ts).
  - `onGet` - **optional**: A one-line JavaScript code to convert the device payload value into the HomeKit-facing value when reading status. The function is called with two arguments: `device` and `value`.
  - `onSet` - **optional**: A one-line JavaScript code to convert the HomeKit-facing value back into the device payload value when sending commands. The function is called with two arguments: `device` and `value`. Returning `undefined` means to skip sending the command.
  - `hidden` - **optional**: Hide the schema. Defaults to `false`.

> 🧭 **Which side should overrides model?** Always describe the HomeKit-facing shape in your overrides. `newCode` and transformed `type`/`property` metadata tell the plugin what HomeKit should see, while `onGet` and `onSet` bridge between that HomeKit view and the raw Tuya payload the device actually uses.

### How overrides are applied

The plugin rewrites schemas and statuses at runtime according to the following flow:

1. **Schema discovery**: When a characteristic asks for a DP, the plugin looks up the matching `schema` override (respecting `newCode` aliases). If a match is marked `hidden`, the DP is ignored; otherwise, the override can change the DP's type or property metadata before HomeKit sees it.
2. **Reading status (`onGet`)**: When status is received from Tuya, the original DP code/value is transformed using `onGet` and/or `newCode` before the accessory handler uses it. This lets you normalize odd Tuya payloads into the shapes HomeKit expects.
3. **Sending commands (`onSet`)**: When HomeKit sends a command, the plugin applies `onSet` to convert the normalized HomeKit value back to the original DP payload, and rewrites the code back to `code` if `newCode` was set.

Because overrides happen dynamically, you can usually adjust behaviour without touching the device cache. However, if you change `category`, `schema` `code`/`newCode`, or `hidden` flags, clear the Homebridge accessory cache to avoid stale metadata.


## Examples

### Change category code

```js
{
  "options": {
    // ...
    "deviceOverrides": [{
      "id": "{device_id}",
      "category": "xxx"
    }]
  }
}
```

### Hide device / scene

Just the same way as changing category code.

```js
{
  "options": {
    // ...
    "deviceOverrides": [{
      "id": "{device_id_or_scene_id}",
      "category": "hidden"
    }]
  }
}
```

### Hide DP

An example of hide camera's floodlight (`floodlight_switch`):
```js
{
  "options": {
    // ...
    "deviceOverrides": [{
      "id": "{device_id}",
      "schema": [{
        "code": "floodlight_switch",
        "hidden": true
      }]
    }]
  }
}
```

### Enable Adaptive Lighting

```js
{
  "options": {
    // ...
    "deviceOverrides": [{
      "id": "{device_id}",
      "adaptiveLighting": true
    }]
  }
}
```


### Offline as off

If you want to display off status when device is offline:
```js
{
  "options": {
    // ...
    "deviceOverrides": [{
      "id": "{device_id}",
      "schema": [{
        "code": "{dp_code}",
        "onGet": "(device.online && value)"
      }]
    }]
  }
}
```


### Change DP code

```js
{
  "options": {
    // ...
    "deviceOverrides": [{
      "id": "{device_id}",
      "schema": [{
          "code": "{old_dp_code}",
          "newCode": "{new_dp_code}"
      }]
    }]
  }
}
```


### Convert from enum DP to boolean DP

An example of convert `open`/`close` into `true`/`false`:
```js
{
  "options": {
    // ...
    "deviceOverrides": [{
      "id": "{device_id}",
      "schema": [{
        "code": "{dp_code}",
        "type": "Boolean",
        "onGet": "(value === 'open') ? true : false;",
        "onSet": "(value === true) ? 'open' : 'close';"
      }]
    }]
  }
}
```


### Adjust integer DP ranges

Some odd thermostat stores double of the real value to keep the decimal part (0.5°C).

We need override both range and value in order to make it working. (Only override value is not enough, range is required too.)

Here's an example of the invalid schema:
```js
{
  code: 'temp_set',
  mode: 'rw',
  type: 'Integer',
  property: { unit: '℃', min: 10, max: 70, scale: 1, step: 5 }
}
```

The value `41` actually represents for `20.5°C`, the range `10~70` actually represents for `5.0°C~35.0°C`.

To fix this, first we need set scale to `1`, and convert `41` to `205` when getting, convert `205` to `41` when getting, which means `value x 5` when getting, and `value / 5` when setting.

Here's the example config:
```js
{
  "options": {
    // ...
    "deviceOverrides": [{
      "id": "{device_id}",
      "schema": [{
        "code": "temp_set",
        "onGet": "(value * 5);",
        "onSet": "(value / 5);",
        "property": {
          "min": 50,
          "max": 350,
          "scale": 1,
          "step": 5
        }
      }]
    }]
  }
}
```

After transform value using `onGet` and `onSet`, and new range in `property`, it should be working now.


### Reverse curtain motor's on/off state

Most curtain motor have "reverse mode" setting in the Tuya App, if you don't have this, you can reverse `percent_control`/`position` and `percent_state` in the plugin config:

```js
{
  "options": {
    // ...
    "deviceOverrides": [{
      "id": "{device_id}",
      "schema": [{
        "code": "percent_control",
        "onGet": "(100 - value)",
        "onSet": "(100 - value)"
      }, {
        "code": "percent_state",
        "onGet": "(100 - value)",
        "onSet": "(100 - value)"
      }]
    }]
  }
}
```


### Skip send on/off command when touching brightness/speed slider

Some products (dimmer, fan) having issue when sending brightness/speed command with on/off command together. Here's an example of skip on/off command.

```js
{
  "options": {
    // ...
    "deviceOverrides": [{
      "id": "{device_id}",
      "schema": [{
        "code": "switch_led",
        "onSet": "(value === device.status.find(status => status.code === 'switch_led').value) ? undefined : value"
      }]
    }]
  }
}
```


### Convert Fahrenheit to Celsius

F = 1.8 * C + 32

C = (F - 32) / 1.8

```js
{
  "options": {
    // ...
    "deviceOverrides": [{
      "id": "{device_id}",
      "schema": [{
        "code": "temp_current",
        "onGet": "Math.round((value - 32) / 1.8);",
        "onSet": "Math.round(1.8 * value + 32);"
      }]
    }]
  }
}
```
