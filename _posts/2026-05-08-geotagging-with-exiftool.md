---
title: "Notes on Geotagging with exiftool"
date: 2026-05-08 21:19:19 +0000
---

### Record a GPS trace from photos with tags:

```
exiftool -i HIDDEN -if '$gps:all' -fileOrder gpsdatetime -GPSDateTime -gpslatitude -gpslongitude -gpsaltitude -GPSLatitudeRef -GPSLongitudeRef -csv /path/to/folder > track.csv
```

### Fix timezone on images to be tagged:

Inspect existing time related data:

```
exiftool -time:all -G1 -a -s /path/to/file
[System]        FileModifyDate                  : 2026:01:01 00:03:02+00:00
[System]        FileAccessDate                  : 2026:04:20 21:09:36+01:00
[System]        FileInodeChangeDate             : 2026:01:01 00:03:02+00:00
[IFD0]          ModifyDate                      : 2026:01:01 00:03:02
[ExifIFD]       DateTimeOriginal                : 2026:01:01 00:03:02
[ExifIFD]       CreateDate                      : 2026:01:01 00:03:02
[Olympus]       DateTimeUTC                     : 2025:12:31 15:03:02
```

In this case we see that the picture has the correct time in UTC, but stored in a proprietary tag (`[Olympus]`). We just need to set a +09:00 timezone offset to make `DateTimeOriginal` correct:

```
exiftool -i HIDDEN '-OffsetTime*=+09:00' /path/to/folder
```

It is critical that the timestamps in the track.csv (`GPSDateTime`, always UTC) and the timestamps in the pictures (`DateTimeOriginal` + `OffsetTime`) are correct. Otherwise exiftool will interpolate from the wrong data points.

### Apply trace:

Adjust `GeoMaxExtSecs` and `GeoMaxIntSecs` depending on the densitiy of you tracking data and how much margin of error is acceptable.

```
exiftool -i HIDDEN -geotag=track.csv -api GeoMaxExtSecs=43200 -api GeoMaxIntSecs=43200 /path/to/folder
```

### Vizualizing a Trace:

For importing a GPS trace into other tools, it's better to use the more common GPX format. This format together with replacing `-csv` with `-p gpx.fmt` does that:

```
#------------------------------------------------------------------------------
# File:         gpx.fmt
#
# Description:  Example ExifTool print format file to generate a GPX track log
#
# Usage:        exiftool -p gpx.fmt -ee3 FILE [...] > out.gpx
#
# Requires:     ExifTool version 10.49 or later
#
# Revisions:    2010/02/05 - P. Harvey created
#               2018/01/04 - PH Added IF to be sure position exists
#               2018/01/06 - PH Use DateFmt function instead of -d option
#               2019/10/24 - PH Preserve sub-seconds in GPSDateTime value
#
# Notes:     1) Input file(s) must contain GPSLatitude and GPSLongitude.
#            2) The -ee3 option is to extract the full track from video files.
#            3) The -fileOrder option may be used to control the order of the
#               generated track points when processing multiple files.
#------------------------------------------------------------------------------
#[HEAD]<?xml version="1.0" encoding="utf-8"?>
#[HEAD]<gpx version="1.0"
#[HEAD] creator="ExifTool $ExifToolVersion"
#[HEAD] xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
#[HEAD] xmlns="http://www.topografix.com/GPX/1/0"
#[HEAD] xsi:schemaLocation="http://www.topografix.com/GPX/1/0 http://www.topografix.com/GPX/1/0/gpx.xsd">
#[HEAD]<trk>
#[HEAD]<number>1</number>
#[HEAD]<trkseg>
#[IF]  $gpslatitude $gpslongitude
#[BODY]<trkpt lat="$gpslatitude#" lon="$gpslongitude#">
#[BODY]  <ele>$gpsaltitude#</ele>
#[BODY]  <time>${gpsdatetime#;my ($ss)=/\.\d+/g;DateFmt("%Y-%m-%dT%H:%M:%SZ");s/Z/${ss}Z/ if $ss}</time>
#[BODY]</trkpt>
#[TAIL]</trkseg>
#[TAIL]</trk>
#[TAIL]</gpx>
```

The resulting GPX can then we viewed with common tools like [gpx.studio](https://gpx.studio).