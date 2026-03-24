# Explainer for the Web Scanning API

This proposal is an early design sketch by the ChromeOS team to describe the problem of document scanning on the web and solicit feedback on the proposed solution. It has not been approved to ship in Chrome.

## Proponents

- Google Chrome

## Participate
- https://github.com/explainers-by-googlers/web-scanning/issues

## Table of Contents

<!-- Update this table of contents by running `npx doctoc README.md` -->
<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Introduction](#introduction)
- [Terminology](#terminology)
- [Goals](#goals)
- [Non-goals](#non-goals)
- [Use cases](#use-cases)
  - [Single-page flatbed scanning](#single-page-flatbed-scanning)
  - [High-volume batch scanning (ADF)](#high-volume-batch-scanning-adf)
  - [Progressive image display](#progressive-image-display)
- [Potential Solution](#potential-solution)
  - [The `ScannerManager` Interface](#the-scannermanager-interface)
  - [The `Scanner` Interface](#the-scanner-interface)
  - [The `ScanJob` Interface](#the-scanjob-interface)
  - [How this solution would solve the use cases](#how-this-solution-would-solve-the-use-cases)
    - [Basic Scan](#basic-scan)
    - [Batch Scan with ADF (Streaming)](#batch-scan-with-adf-streaming)
- [Detailed design discussion](#detailed-design-discussion)
  - [User-Mediated Device Selection (The Chooser Pattern)](#user-mediated-device-selection-the-chooser-pattern)
  - [Streaming vs. Atomic Results](#streaming-vs-atomic-results)
  - [Normalized Coordinate Geometry](#normalized-coordinate-geometry)
- [Considered alternatives](#considered-alternatives)
  - [Direct Hardware Access (WebUSB/WebBluetooth)](#direct-hardware-access-webusbwebbluetooth)
  - [Extension-based APIs](#extension-based-apis)
- [Security and Privacy Considerations](#security-and-privacy-considerations)
- [Stakeholder Feedback / Opposition](#stakeholder-feedback--opposition)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Introduction

Web applications requiring document scanning currently rely on fragmented, non-standard approaches, such as proprietary browser extensions or legacy app models (e.g., Chrome Apps using the [`chrome.documentScan`](https://developer.chrome.com/docs/extensions/reference/api/documentScan) API). These legacy models tightly couple web applications to specific browser ecosystems or require users to install heavy, OS-specific native companion applications. These methods severely limit cross-browser compatibility, create maintenance burdens for developers, and pose security risks as browser vendors transition away from extension-bound hardware access (such as the shift to Manifest V3).

The Web Scanning API provides a standardized, secure, and web-native interface for document scanning. It abstracts underlying, highly varied hardware protocols into a cohesive API. These protocols include:
- **TWAIN / TWAIN Direct:** A legacy driver-based standard and its modern RESTful network successor.
- **eSCL (AirScan):** A driverless, network-based scanning protocol developed by the Mopria Alliance.
- **SANE (Scanner Access Now Easy):** A standardized API predominantly used in UNIX/Linux environments.
- **WSD (Web Services for Devices):** Microsoft's native architecture for network-connected devices.

By abstracting these protocols, the API ensures that web applications can interact with scanning hardware through a uniform, user-mediated permission model without needing to understand the underlying transport.

## Terminology

To help developers without background in imaging hardware, we define the following terms:
- **Platen (Flatbed):** The glass surface where a document is placed manually. Best for single pages or fragile documents.
- **ADF (Automatic Document Feeder):** A component that automatically pulls a stack of paper into the scanner. Essential for high-volume batch scanning.
- **Duplex Scanning:** The ability to scan both sides of a piece of paper automatically.
- **Resolution (DPI):** Measured in Dots Per Inch. Higher DPI means more detail but larger file sizes and slower scan speeds.
- **Color Mode:** Options typically include `color` (24-bit), `grayscale` (8-bit), and `monochrome` (1-bit black and white).
- **Paper Jam:** A mechanical error where paper becomes stuck inside the ADF mechanism.

## Goals

- **Standardization:** Provide a unified API for document scanning agnostic of the underlying platform.
- **Hardware Abstraction:** Hide the complexity of various scanning protocols (TWAIN, eSCL, etc.) behind a clean JavaScript interface.
- **Security & Privacy:** Ensure hardware access is strictly controlled by the user through a permission-gated device picker, preventing silent surveillance or fingerprinting.
- **Efficiency:** Use streaming to handle high-resolution or multi-page scans without exhausting system memory.
- **Asynchronicity:** Handle mechanical hardware latency using Promises, events, and asynchronous iterables.

## Non-goals

- **Low-level Driver Access:** The API is not intended to provide raw access to hardware registers or proprietary driver settings that are not relevant to standard scanning tasks.
- **Direct Network Discovery:** Web pages will not be able to silently discover all scanners on a local network; this discovery is managed by the browser during the device selection process.

## Use cases

### Single-page flatbed scanning
A user needs to scan a single document (e.g., a signed contract) from a flatbed scanner into a web-based document management system.

### High-volume batch scanning (ADF)
A user needs to scan a stack of documents using an Automatic Document Feeder (ADF), potentially with duplex (two-sided) scanning, into a medical or enterprise record system.

### Progressive image display
A user scans a high-resolution photo. To ensure the UI feels responsive, the application displays chunks of the image as they are received from the scanner, rather than waiting for the entire multi-megabyte file to finish.

## Potential Solution

The API is exposed via `navigator.scanners` and uses a pattern similar to WebUSB or WebHID, where a user-mediated chooser is the primary entry point for hardware access.

### The `ScannerManager` Interface
The entry point for the API, used to request access to scanners.

```js
// Triggers a browser-native device picker.
const scanner = await navigator.scanners.requestScanner({
  filters: [{ vendorId: '0x04a9' }] // Optional filtering
});
```

### The `Scanner` Interface
Represents a specific scanning device authorized by the user. Identifiers are randomized and scoped to the origin.

```js
const capabilities = await scanner.getCapabilities();
console.log(`Available resolutions: ${capabilities.resolutions}`);

const controller = new AbortController();
const job = await scanner.scan({
  resolution: 300,
  colorMode: 'color',
  source: 'adf',
  signal: controller.signal
});
```

### The `ScanJob` Interface
Represents an ongoing scan operation. It implements `AsyncIterable` to yield pages as they complete.

```js
// The job provides streaming access to scanned pages.
for await (const page of job) {
  console.log(`Received page ${page.index}`);
  const blob = await page.getBlob();
  // page also provides a ReadableStream for chunked data
  const stream = page.data; 
}

job.onpaperjam = () => {
  console.error("The ADF is jammed. Please clear the paper path.");
};
```

### How this solution would solve the use cases

#### Basic Scan
```js
async function performSimpleScan() {
  const scanner = await navigator.scanners.requestScanner();
  const job = await scanner.scan({ resolution: 300 });
  
  // For a simple scan, we just take the first result
  const { value: firstPage } = await job[Symbol.asyncIterator]().next();
  const blob = await firstPage.getBlob();
  
  const img = document.createElement('img');
  img.src = URL.createObjectURL(blob);
  document.body.appendChild(img);
}
```

#### Batch Scan with ADF (Streaming)
```js
async function performBatchScan() {
  const scanner = await navigator.scanners.requestScanner();
  const job = await scanner.scan({
    source: 'adf',
    duplex: true
  });
  
  for await (const page of job) {
    await uploadPage(await page.getBlob());
    console.log(`Uploaded page ${page.index}`);
  }
  
  console.log("All pages scanned and uploaded.");
}
```

## Detailed design discussion

### User-Mediated Device Selection (The Chooser Pattern)
To prevent fingerprinting and unauthorized hardware access, the API mandates a browser-native device chooser. Web applications cannot enumerate all connected scanners without an explicit user gesture. The `requestScanner()` method triggers this UI, and the resulting `Scanner` object represents a specific user-granted permission.

### Streaming vs. Atomic Results
Scanning a single A4 page at 600 DPI can produce over 100MB of raw data. Returning this as a single atomic `Blob` can lead to out-of-memory crashes, especially on mobile devices or low-end hardware. The `ScanJob` interface uses `ReadableStream` and `AsyncIterable` to allow developers to process data as it arrives, enabling progressive rendering and efficient uploads.

### Normalized Coordinate Geometry
Different scanning protocols use different units (e.g., eSCL uses 1/300ths of an inch). The Web Scanning API normalizes all spatial dimensions to **millimeters** (physical) or **CSS pixels** (logical), ensuring a consistent developer experience across all hardware.

## Considered alternatives

### Direct Hardware Access (WebUSB/WebBluetooth)
While WebUSB/WebBluetooth provide low-level access, they require web developers to implement complex protocol drivers (e.g., TWAIN over USB) in JavaScript. This is extremely burdensome, prone to errors, and insecure as it exposes raw hardware buffers to the web page.

### Extension-based APIs
Legacy APIs like `chrome.documentScan` are restricted to specific browser environments and extensions.

## Security and Privacy Considerations

- **Secure Contexts:** The API is strictly limited to HTTPS origins.
- **Permissions Policy:** Access is controlled by the `web-scanning` policy, defaulting to `self`.
- **Randomized Identifiers:** Scanner IDs are randomized and origin-bound. `Scanner.id` on `example.com` will be different from the ID for the same device on `other-site.com`.
- **Transient Activation:** Requests for hardware access (`requestScanner`) or starting a scan (`scan`) require a user gesture (e.g., a click).
- **Visible Indicators:** Browsers should show a persistent "Scanning..." indicator while the hardware is active, similar to camera/microphone indicators.

## Stakeholder Feedback / Opposition

- **Google (Chrome):** Positive, driving the proposal.
- **TWAIN Working Group:** Potentially positive, as their TWAIN Direct standard is explicitly designed to enable scanners to communicate directly with web applications ([TWAIN Direct:™ Easy Scanning for Every Application](https://www.info-source.com/twain-direct-easy-scanning-for-every-application/)).
- **Mopria Alliance (eSCL):** Potentially positive, as the proposed API leverages their vendor-neutral driverless standard ([Mopria eSCL Specification Download](https://mopria.org/spec-download)).


