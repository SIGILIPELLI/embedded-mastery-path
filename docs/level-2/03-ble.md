# Bluetooth Low Energy (BLE)

WiFi (module 1-09) is great when there's an access point and you don't mind
the power draw. **Bluetooth Low Energy** is the other half of the ESP32's
radio: no router needed, milliamps instead of tens of milliamps, and a phone
in every pocket already knows how to speak it. This module covers the two
concepts BLE is built from — **GAP** and **GATT** — then builds a BLE
peripheral a phone app (nRF Connect, LightBlue) can discover, read, write to,
and receive live notifications from.

## GAP: how devices find each other

**GAP** (Generic Access Profile) governs discovery and connection, before
any data is exchanged. A device plays one of two roles here:

- **Peripheral** — advertises its presence and accepts connections (your
  ESP32, in this module).
- **Central** — scans for advertisements and initiates connections (the
  phone).

Advertising is a small broadcast packet sent repeatedly (name, a service
UUID or two, maybe a manufacturer field) that any scanning central can see
without connecting — this is how a phone's Bluetooth settings screen lists
devices before you tap one.

## GATT: how data is structured

Once connected, **GATT** (Generic Attribute Profile) defines the data model:

| Level | What it is |
|-------|------------|
| **Service** | A group of related functionality, identified by a UUID (e.g. "Environmental Sensing") |
| **Characteristic** | One value inside a service (e.g. "Temperature"), with its own UUID, a value, and properties |
| **Properties** | What's allowed: `READ`, `WRITE`, `NOTIFY`, `INDICATE`, ... |
| **Descriptor** | Metadata attached to a characteristic — most importantly the **CCCD** (`0x2902`), which a central writes to *subscribe* to notifications |

UUIDs are 128-bit (or a handful of Bluetooth SIG-reserved 16-bit ones) —
generate your own with any UUID v4 generator; they just need to be unique to
your service, not registered anywhere.

## Building a BLE peripheral

Arduino-ESP32's `BLEDevice` library wraps the underlying NimBLE/Bluedroid
host stack in a small set of classes that map directly onto the GAP/GATT
concepts above:

```cpp
#include <BLEDevice.h>
#include <BLEServer.h>
#include <BLEUtils.h>
#include <BLE2902.h>

#define SERVICE_UUID        "4fafc201-1fb5-459e-8fcc-c5c9c331914b"
#define CHARACTERISTIC_UUID "beb5483e-36e1-4688-b7f5-ea07361b26a8"

BLECharacteristic *pCharacteristic;
bool deviceConnected = false;

class MyServerCallbacks : public BLEServerCallbacks {
  void onConnect(BLEServer *pServer) override {
    deviceConnected = true;
    Serial.println("central connected");
  }
  void onDisconnect(BLEServer *pServer) override {
    deviceConnected = false;
    Serial.println("central disconnected — advertising again");
    pServer->getAdvertising()->start();   // advertising stops on connect; restart it
  }
};

class MyCharacteristicCallbacks : public BLECharacteristicCallbacks {
  void onWrite(BLECharacteristic *c) override {
    std::string value = c->getValue();
    if (!value.empty()) {
      Serial.printf("central wrote: %s\n", value.c_str());
    }
  }
};

void setup() {
  Serial.begin(115200);

  BLEDevice::init("ESP32-Sensor");                    // sets the advertised name
  BLEServer *pServer = BLEDevice::createServer();
  pServer->setCallbacks(new MyServerCallbacks());

  BLEService *pService = pServer->createService(SERVICE_UUID);

  pCharacteristic = pService->createCharacteristic(
    CHARACTERISTIC_UUID,
    BLECharacteristic::PROPERTY_READ  |
    BLECharacteristic::PROPERTY_WRITE |
    BLECharacteristic::PROPERTY_NOTIFY
  );
  pCharacteristic->addDescriptor(new BLE2902());       // CCCD — required for notify()
  pCharacteristic->setCallbacks(new MyCharacteristicCallbacks());
  pCharacteristic->setValue("0.0");

  pService->start();

  BLEAdvertising *pAdvertising = BLEDevice::getAdvertising();
  pAdvertising->addServiceUUID(SERVICE_UUID);
  pAdvertising->start();
  Serial.println("advertising as ESP32-Sensor");
}

void loop() {
  if (deviceConnected) {
    static float t = 20.0f;
    t += 0.1f;
    char buf[16];
    snprintf(buf, sizeof(buf), "%.1f", t);
    pCharacteristic->setValue(buf);
    pCharacteristic->notify();          // pushed only to centrals that subscribed
  }
  delay(2000);
}
```

Open **nRF Connect** (Android/iOS) or **LightBlue**, scan, connect to
`ESP32-Sensor`, and the characteristic appears under its UUID — read it,
write a value back (watch it print over serial), and enable notifications
(the app writes the CCCD for you) to watch the number tick every 2 s.

!!! warning "`notify()` with nobody subscribed does nothing — silently"
    Calling `pCharacteristic->notify()` before a central has written its
    CCCD is not an error; the stack just drops it. If notifications "aren't
    arriving," check the central actually subscribed (in nRF Connect, tap
    the download-arrow icon next to the characteristic) before suspecting
    the firmware.

## Traps worth knowing before you build further

- **Advertising stops on connection.** A peripheral advertises to attract a
  connection, then stops — only one central can be connected at a time on
  the classic single-connection model. `onDisconnect()` must explicitly
  restart advertising, or the device becomes unreachable after the first
  phone disconnects (exactly the bug the callback above avoids).
- **MTU is small by default** — 23 bytes total, ~20 usable per
  characteristic write/notify, until the central negotiates a larger MTU.
  Don't assume you can shove a JSON blob through in one notification without
  checking negotiated size or chunking it.
- **BLE and WiFi share one radio and one 2.4 GHz antenna.** Running both
  simultaneously (module 2-10's project territory) works, but throughput on
  each drops under contention — budget for it rather than being surprised.
- **The BLE stack costs real heap** — tens of KB just to bring the stack up,
  on top of whatever WiFi already reserved. On memory-constrained builds
  this is a genuine design constraint, not a rounding error.

## Cheat sheet

| Concept | Detail |
|---------|--------|
| GAP | Discovery/connection layer — roles: peripheral (advertises) / central (scans) |
| GATT | Data layer — services contain characteristics; characteristics have properties |
| Properties | `READ`, `WRITE`, `NOTIFY`, `INDICATE` — what a central may do |
| CCCD (`0x2902`) | Descriptor a central writes to subscribe to notify/indicate |
| `BLEDevice::init(name)` | Sets the advertised device name, starts the BLE stack |
| `createServer()` → `createService()` → `createCharacteristic()` | GATT tree, built top-down |
| `BLE2902` descriptor | Required on any `NOTIFY`/`INDICATE` characteristic |
| `pCharacteristic->notify()` | Push value to subscribed centrals only — no-op if none |
| `onDisconnect()` | Must call `pServer->getAdvertising()->start()` to be reachable again |
| Default MTU | 23 bytes (~20 usable) unless negotiated up |

## Exercise

Extend the peripheral above with a second, writable-only characteristic
(`PROPERTY_WRITE`) that accepts `"on"`/`"off"` and drives the onboard LED
from `onWrite()`, plus a read-only characteristic reporting
`ESP.getFreeHeap()` as a string, refreshed each time it's read (override
`onRead()`). Connect with nRF Connect, verify: the LED responds to writes,
the heap characteristic's value changes between reads, and the notify
characteristic keeps ticking. Then disconnect deliberately and confirm the
device reappears in a new scan within a few seconds — proof your
`onDisconnect()` handler is doing its job.
