---
title: "Testing ISO Invariance of the Olympus E-M10 II"
date: 2026-10-26
---

![camera](/assets/images/iso-invariance-em10ii/em10ii.jpg)

My Olympus E-M10II is advertised as a 200 to 5000 ISO camera, with values above and below being "extensions". Yet I read many times that ISOs above so and so are unusable or garbage (usually 3200 or 6400). Let's see if we can figure out some more with statistics.

Most of the math here is based on [photonstophotos.net - Sensor Analysis Primer - Read Noise - Optical Black](https://www.photonstophotos.net/GeneralTopics/Sensors_&_Raw/Sensor_Analysis_Primer/Read_Noise_-_Optical_Black.htm). Give it a read, it's pretty informative.

That being said, I have no clue what I'm doing.

## Background

Photons go though many stages before getting stored in a raw file. Many of those steps are not perfect and introduce noise. Here is a rough overview:

![iso_pipeline](/assets/images/iso-invariance-em10ii/iso_pipeline.png)

We can really only control the input and observe the final output, the intermediate results are hidden inside the camera. But, with some clever math and physics, some information can be deduced.

## Preparation

**Data:** You will need at least one raw black frame (lens cap on) per major ISO setting (LOW, 200, 400, 800, etc). Turn *Noise Reduction* (different from *Noise Filter*!) and *Shading Compensation* off as they mess with the raw data. Multiple shots don't hurt if you want more statistical significance. Shutter speed does not really matter, but don't do long exposures as it makes hot pixels worse by heating up the sensor. Pick 1/100 and stick with it. Lens and Aperture do not matter, we're not capturing any light (I guess a body cap would be best for this test?).

numpy and rawpy to do the heavy lifting:

```python
import numpy as np, rawpy

# raw values of one green channel
def green(path):
    with rawpy.imread(path) as raw:
        return raw.raw_image_visible[0::2, 1::2].ravel().astype(int)

# standard deviation without hot pixels
def noise(x):
    med = np.median(x)
    mad = np.median(np.abs(x - med))
    return x[np.abs(x - med) < 7.4 * mad].std()  # 5σ
```

*Why only one green channel? Red, blue and the second green gave me the same results, this keeps the code simple.*

## Test 1: Which Raw Values Occur?

Read noise spreads a black frame over the values around the black level. An analog gain
widens that spread and still produces every value. Multiplying integers by *k* puts them on a grid
with steps of *k*, so a share of *1 − 1/k* of the values stays empty.

```python
g = green("lens_cap_iso_5000.ORF")
counts = np.bincount(g)  # histogram with one bin per raw value
mu, sd = g.mean(), noise(g)  # center and spread
v = np.arange(int(mu - 2 * sd), int(mu + 2 * sd) + 1)  # we're only looking at the core of the histogram, 2σ
for offset, n in zip(v - round(mu), counts[v]):  # relative to center
    print(f"{offset:+4d} {n:8d}")
```

This gives us:

```
 -20        0
 -19    38989
 -18    52575
 -17        0
 -16    68884
 -15        0
 -14    88145
 -13   110907
 -12        0
 -11   134651
 -10   159997
  -9        4
  -8   185031
  -7       17
  -6   208281
  -5   228684
  -4       34
  -3   243926
  -2   254485
  -1       32
  +0   258715
  +1       28
  +2   256488
  +3   245961
  +4       18
  +5   231683
  +6   211104
  +7       11
  +8   188362
  +9   163978
 +10        4
 +11   139408
 +12        2
 +13   114958
 +14    92379
 +15        0
 +16    72675
 +17    55338
 +18        0
 +19    41090
```

Aha! So some raw values never (or very rarely) occur in the 5000 ISO file! Lets repeat this for the other ISO values and plot them for easier reading. Here are the results:

![Histograms of raw values near black at ISO 1600, 3200, 5000, 12800 and 25600](/assets/images/iso-invariance-em10ii/histograms.png)

| ISO | empty values | expected if ISO 3200 × *k* |
|---:|---:|---:|
| 200–3200 | 0% | – |
| 5000 | 37.6% | 36.9% |
| 12800 | 75.8% | 75.0% |
| 25600 | 87.6% | 87.5% |

ISO 3200 is the highest ISO that still fills every value. Every higher setting just digitally multiplies (i.e. brightens) the sensor readings without adding more signal.

## Test 2: Noise vs Gain

A higher analog gain should lift the signal above the noise added after the amplifier, so noise referred back to the sensor should drop. A digital multiply on the other hand just scales signal and noise alike: the black-frame noise grows by exactly *k*.

We already have everything to plot the various noise DN values:
```python
dn = noise(green("lens_cap_iso_5000.ORF"))  # Digital Numbers
```

Then, calculate the relative noise increase for all ISO steps:

```python
noise(green("lens_cap_iso_5000.ORF")) / noise(green("lens_cap_iso_3200.ORF"))   # 1.589
```

![Black-frame noise per ISO, as recorded and divided by the gain](/assets/images/iso-invariance-em10ii/noise.png)


| step | noise factor | ISO factor | noise per ISO |
|---|---:|---:|---:|
| 200 → 400 | ×1.110 | ×2 | 0.555 |
| 400 → 800 | ×1.720 | ×2 | 0.860 |
| 800 → 1600 | ×1.821 | ×2 | 0.910 |
| 1600 → 3200 | ×1.876 | ×2 | 0.938 |
| 3200 → 5000 | ×1.589 | ×1.5625 | 1.017 |
| 5000 → 12800 | ×2.577 | ×2.56 | 1.007 |
| 12800 → 25600 | ×1.958 | ×2 | 0.979 |

Up to 3200, each doubling of ISO raises the noise by less than ×2. Above 3200 the noise roughly keeps pace with the ISO. On its own, noise is a weak test. The empty values from test 1 are more definitive.


## Conclusion

- **ISO 3200 is the E-M10II's highest analog ISO.** ISO 2000, 2500, and below are analog too.
- **Every ISO above it is ISO 3200 digitally multiplied and rounded.** The noise stays the same, values go missing, and the highlights clip
  1, 2 and 3 stops earlier at 6400, 12800 and 25600.
- **In practice:** cap *Auto ISO* at 3200 and brighten in post. Except for SOOC JPGs (which would be unusably dark), there are only downsides.
