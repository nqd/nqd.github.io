---
layout: post
title: JSON object flatten/unflatten in Golang
description: JSON object flatten/unflatten package in golang
published: true
---

One of the new things that Amazon IoT adding above MQTT is the shadow doc. Shadow doc provides a traditional way (aka HTTP) to get the current status of a device that connects via MQTT.

Basic function of a shadow doc is the ability to update part of its JSON object. For example, one thermostat has shadow `{  "ambient": 25.1 }`, then has the setPoint updated with `{ "setPoint": { "cool": 20.5 } }` to get combined doc:

```go
{
  "ambient": 25.1,
  "setPoint": {
    "cool": 20.5
  }
}
```

It can be achieved with JSON supported databases, like Postgresql or MongoDB. Before that, I need to flatten the JSON object.

There is good package in JS [flat](https://github.com/hughsk/flat) that serve the purpose well. I used it with [aedes](https://github.com/mcollina/aedes/) as a small clone of AWS IoT pretty good.

In Golang, package [flatten](https://github.com/jeremywohl/flatten) serve for the same purpose. I find it lacks options to keep the slice, the max deep option to be compatible with AWS IoT shadow doc. This leads me to release the flat for Golang at [https://github.com/nqd/flat](https://github.com/nqd/flat).

This simple package is enough for me to build a shadow doc, though there  are some more options for the package to make it more "complete":

- Safe option for Unflatten
- Overwrite.