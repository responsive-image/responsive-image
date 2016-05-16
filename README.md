# Responsive Image

This script works as drop in, considering few assumptions.

## Solution & Assumptions

1. `@1x` also corresponds to the size of an image @ 320 device
   points dimension, which is considered `@1x` image pixel ratio (IPR)
2. server-side generation of thumbnails/sizes is in sync with
   the assumption #1
3. Image picking depends on:
   - network speed
   - device pixel ratio (DPR)
   - area an image should fit into

## Image Pixel Ratio - IPR

**Image Pixel Ratio** (IPR) is an artificial term that serves our simplification purposes. It is a value that means how many times an image would fit into square of 320x320 pixels.

Base of 320 pixels we calculate IPR is selected to make sure an image tagged as @1x would cover the screen area of size 320x320 px. Due to portrait/landscape orientation we always need to consider the larger screen (or any other are image should fill) dimension to calculate the IPR.

Ratio precision is always 1 decimal place max, rounded downwards.

Formula to determine IPR of an image stored on server:

```javascript
image.ipr = Math.round(Math.max(image.width, image.height) / 320 * 10) / 10;
```

Example image sizes to IPR:

- 320 x 240 &rarr; max = 332 &rarr; 320 / 320 = @1x IPR
- 640 x 480 &rarr; max = 480 &rarr; 480 / 320 = @1.5x IPR
- 1920 x 1280 &rarr; max = 1920 &rarr; 1920 / 320 = @6x IPR

or any miscellaneous image:

- 560 x 2480 &rarr; max = 2480 &rarr; 2480 / 320 = @2.7x IPR

## Relation of IPR to DPR

A device has a known screen width and height and device pixel ratio (DPR). Screen size defines the maximum size of image that can be displayed.

A devices with larger device pixel area might require image with more IPRs to display image than small device with higher DPR.

Formula to find out required IPR to request:

```js
img.ipr = Math.max(img.width, img.height) / 320 * dpr;
```

Example IPR to request for different image sizes @ various DPRs:

- 320 x 240 @1x DPR &rarr; max = 320 &rarr; 320 * 1xDPR = 320 &rarr; 320 / 320 = @1xIPR
- 320 x 240 @2x DPR &rarr; max = 320 &rarr; 320 * 2xDPR = 640 &rarr; 640 / 320 = @2xIPR
- 640 x 480 @1x DPR &rarr; max = 640 &rarr; 640 * 1xDPR = 640 &rarr; 640 / 320 = @2xIPR
- 640 x 480 @2x DPR &rarr; max = 640 &rarr; 640 * 2xDPR = 1280 &rarr; 1280 / 320 = @4xIPR
- 640 x 480 @3x DPR &rarr; max = 640 &rarr; 640 * 3xDPR = 1920 &rarr; 1920 / 320 = @6xIPR

and a fairly large screen would require same image size:

- 1920 x 1 080 @1x DPR &rarr; max = 192 &rarr; 192 * 1xDPR = 1920 &rarr; 1920 / 320 = @6xIPR


## Network Speed Ratio (SPR)

At ideal network latency, all images should load in respective IPR size at current device pixel ratio. However the speed of the network connection varies, therefore to ensure good user experience, sometimes we want to serve lower quality images on purpose.

Base of 1 SPR for our calculation is speed of a good 2G network:

- latency: 300 ms
- speed: 250kb/s download

To request an image of 1xIPR (320x320 px) from server to display on device with 1xDPR (320x320 physical pixels) from server would require a speed of of 1xSPR at least. Anything below the base would result in loading lower quality image.

To request image on device with 2x higher DPR, would require an image of 2xIPR and woud requre 4xSPR to load. 4xSPR translates to a good 3G connection (1.5Mb/s) to serve same physical size image @2x retina display.

Few rules:

1. IPR size of images are requested at max IPR of the device screen size * DPR;
2. Loading higher density image is an IPR ^ 2 problem;

## Image Quality Ratio (IQR)

To overcome NPR limitations an IQR based selection could be implemented which is useful when serving photo assets. If resource with lower quality is available, than is picked instead of higher quality one. IQR must be applied on both client and server.

IQR compensates IPR for slow SPR scenarios. Base of JPEG quality of 75 is considered 1IQR;

Formula:

```js
lowqualityimage.iqr = (image.filesize / lowqualityimage.filesize) / image.ipr
```

- 320x320 @ 1xIQR ~ 1xIPR (Hight quality image)
- 320x320 @ 2xIQR ~ 2xIPR (Low quality image)

## Srcset Format

The `srcset` format is custom, but tries to stay as close to original W3C standard as possible.

Srcset is comma separated list of resources. If `1x` is assumed if ommited.

Examples:

- `image.jpg, retina-image.jpg 2x, low-quality-retina-image.jpg 2q`

Scenarios:

- `image.jpg`
  - fits 320x320px ~ 1xIPR * 1xDPR * 1xSPR * 1xIQR = 1
  - fits 320x320px ~ 1xIPR * 2xDPR * 0.5xSPR * 1xIQR = 1
  - fits 640x640px ~ 2xIPR * 1xDPR * 0.5xSPR * 1xIQR = 1
- `retina-image.jpg 2x`
  - fits 320x320px ~ 1xIPR * 2xDPR * 1xSPR * 1xIQR = 2
  - fits 640x640px ~ 2xIPR * 1xDPR * 1xSPR * 1xIQR = 2
  - fits 1280x1280 ~ 3xIPR * 1xDPR * 0.33xSPR * 1xIQR = 2
- `low-quality-retina-image.jpg 2q`
  - fits 160x160px ~ 0.5xIPR * 1xDPR * 1xSPR * 2xIQR = 2
  - fits 160x160px ~ 0.5xIPR * 1xDPR * 0.5xSPR * 2xIQR = 2
  - fits 320x320px ~ 1xIPR * 2xDPR * 0.5xSPR * 2xIQR = 2
  - fits 640x640px ~ 2xIPR * 1xDPR * 0.5xSPR * 2xIQR = 2
  - fits 1280x1280 ~ 3xIPR * 1xDPR * 0.33xSPR * 2xIQR = 2
