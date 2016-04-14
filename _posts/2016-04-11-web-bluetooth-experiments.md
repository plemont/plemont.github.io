---
layout: post
title:  "Web Bluetooth Experiments"
date:   2016-04-11 15:39:48 +0100
categories: javascript bluetooth
---
## Introduction

At a recent hackathon I got the chance to experiment with [Web Bluetooth](https://webbluetoothcg.github.io/web-bluetooth/). The specification abstract states Web Bluetooth is an:

*"...API to discover and communicate with devices over the Bluetooth 4 wireless standard using the [Generic Attribute Profile (GATT)](https://developer.bluetooth.org/TechnologyOverview/Pages/GATT.aspx)."*

Being even less familiar with Bluetooth than most technologies, this ultimately ended up meaning: "*will work with some (newer) Bluetooth peripherals, but likely not with many older ones*".

As part of my initial experimentation at the hackathon, I had tried connecting to my [Polar M400 watch](http://www.polar.com/uk-en/products/sport/M400) - some limited success. However, it was my hunch that interacting fullly with the watch was not possible using GATT, and that it likely relied on an older standard, incompatible with Web Bluetooth. (*to be explored...*)

## Polar H7

On returning home I reasoned that my [Polar H7 sensor](http://www.polar.com/uk-en/products/accessories/H7_heart_rate_sensor) was likely to be a much better candidate for meaningful use with Web Bluetooth:

*   It is a [Bluetooth LE](https://en.wikipedia.org/wiki/Bluetooth_low_energy) (BLE) device.
*   BLE provides most of its functionality through key/value pairs provided
    by GATT.
*   GATT already defines a [`heart_rate` service](https://developer.bluetooth.org/gatt/services/Pages/ServiceViewer.aspx?u=org.bluetooth.service.heart_rate.xml), so for maximum compatibility
    this would seem the logical choice.

## Setup

At present, support for the API is not widespread and not generally complete. If the API is available within your browser, then the [`navigator.bluetooth`](https://webbluetoothcg.github.io/web-bluetooth/#dom-navigator-bluetooth) object will exist. When using the API in application, it would be worth testing whether this exists.

On both Chrome OS and Android 6 it was necessary to enable Web Bluetooth from the [chrome://flags](chrome://flags) page. I was not able to get Ubuntu 16.04 with Chrome 50 Beta working at the time.

## Initial Connection

To connect to a device, the [`navigator.bluetooth.requestDevice(options)`](https://webbluetoothcg.github.io/web-bluetooth/#dom-bluetooth-requestdevice) function is used. Through this, filters can be set which determine which devices are returned. These are presented to the user via the user-interface, and user interaction and selection of the device to connect to is a fundamental part of the security of the specification.

The `options` passed to `requestDevice` allow [devices to be filtered by](https://webbluetoothcg.github.io/web-bluetooth/#matches-a-filter):

*   Name
*   Name prefix
*   [Service(s)](https://developer.bluetooth.org/gatt/services/Pages/ServicesHome.aspx) required

So, it would be possible to specify that I am interested in any devices offering the `heart_rate` service.

However, as I am only interested in my Polar device, I chose using a name prefix, and added the `heart_rate` service also:

```javascript
let options = {
  filters: [{
    services: [0x180D], // heart_rate service
    namePrefix: "Polar H7"
  }]
};
navigator.bluetooth.requestDevice(options).then(/* ... */)
```

When successful, the returned Promise resolves to a [`BluetoothDevice`](https://developer.mozilla.org/en-US/docs/Web/API/BluetoothDevice).

`BluetoothDevice` has a single method [`connectGATT`](https://developer.mozilla.org/en-US/docs/Web/API/BluetoothDevice/connectGATT) to facilitate connecting to the selected device.

## Services and Characteristics

A successful call of `connectGATT` returns a Promise that resolves to a [`BluetoothRemoteGATTServer`](https://developer.mozilla.org/en-US/docs/Web/API/BluetoothRemoteGATTServer). This represents the GATT server on the Polar H7.

The server provides access to services and characteristics: [*Services*](https://learn.adafruit.com/introduction-to-bluetooth-low-energy/gatt#services) group logical collections of capabilities known as [*characteristics*](https://learn.adafruit.com/introduction-to-bluetooth-low-energy/gatt#characteristics).

For example, the `heart_rate` service has several characteristics, such as [`heart_rate_measurement`](https://developer.bluetooth.org/gatt/characteristics/Pages/CharacteristicViewer.aspx?u=org.bluetooth.characteristic.heart_rate_measurement.xml) and [`body_sensor_location`](https://developer.bluetooth.org/gatt/characteristics/Pages/CharacteristicViewer.aspx?u=org.bluetooth.characteristic.body_sensor_location.xml).

Once connected, the fun begins, as the services and characteristics of the device can be used to access individual values. The following is an example of obtaining a specific characteristic, once successfully connected:

```javascript
/* from connectGATT() ... */
.then(server => {
  return server.getPrimaryService(0x180D); // heart_rate
})
.then(service => {
  return service.getCharacteristic(0x2A37); // heart_rate_measurement
})
.then(characteristic => /* ... handle characteristic ... */)
```

## Monitoring Changes to Characteristics

Having obtained a `BluetoothRemoteGATTCharacteristic` representing the characteristic, [`readValue`](https://developer.mozilla.org/en-US/docs/Web/API/BluetoothRemoteGATTCharacteristic/readValue) can be used to obtain a Promise that will ultimately resolve to a value.

However, in the case of the heart rate monitor, I want to keep monitoring for changes to the value. For this, the [`startNotificatons`](https://developer.mozilla.org/en-US/docs/Web/API/BluetoothRemoteGATTCharacteristic/startNotifications) method is available. A function can be passed to this method, which will be called with the new value whenever a change occurs.

For example:

```javascript
characteristic.addEventListener('characteristicvaluechanged',
    event => this.onHeartRateChanged_(event));
return characteristic.startNotifications();
```

## Putting it together

![Screenshot]({{ site.url }}images/2016-04-14.png){:width="250px"}

[Example application](https://plemont.github.io/apps/hrm/heartRate.html) - [source](https://github.com/plemont/plemont.github.io/tree/master/apps/hrm)
