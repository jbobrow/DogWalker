# DogWalker
Using an AirTag to save the locations our pup 🐕‍🦺 visits when on his walks.

## Summary
This project uses the fact that a mac computer with FindMy installed will keep a cache of the latest AirTag location while the application is running. Therefore, the following is the my plan for getting the AirTags historical data.
1. Locate AirTag Cache
2. Write Bash Script to parse JSON in Cache and save Lat,Lon data to CSV
3. Create Automator App to run Bash Script
4. Schedule a task using `launchd` and a .plist to run the App every 5 minutes
5. Set FindMy to run on launch

## AirTag Cache Location
Thanks to [someone else doing the exact same thing](https://www.arecadata.com/apple-airtag-location-history-data-pipeline-dog-tracking-dashboard/) (although way more confusing for me), the cache lives at the following path:
> [!IMPORTANT]
>`{HOME}/Library/Caches/com.apple.findmy.fmipcore/Items.data`

## Bash Script to Parse JSON
To create a script that parses the JSON, we first need to understand the contents of the JSON (or at very least the format/nesting).

### AirTag Cache JSON Format
The following is a summary of the JSON data that we will be using. The JSON is organized into an ARRAY of objects(AirTags) and each one contains the `name`, `location`, and `crowdSourcedLocation`. Inside of `location` and `crowdSourcedLocation` are the `latitude`, `longitude`, as well as the `timeStamp` from when that value was received. You can think of location as values confirmed by your device and the crowd-sourced location as a value confirmed by someone else's device. There is a lot of other information in this JSON and we will not be using any of it.

  #### AirTag Object
  - __name__ 
  - __location__
    - timeStamp
    - latitude
    - longitude 
  - __crowdSourcedLocation__
    - timeStamp
    - latitude
    - longitude 

<details>
<summary><h3>Example JSON</h3></summary>
  
```
[{
    "batteryStatus" : 1,
    "isAppleAudioAccessory" : false,
    "partInfo" : null,
    "capabilities" : 766,
    "address" : null,
    "role" : {
      "emoji" : "🐕‍🦺",
      "identifier" : 999,
      "name" : "Custom Name"
    },
    "serialNumber" : "############",
    "safeLocations" : [
      {
        "name" : "Home",
        "address" : {
          "country" : "United States",
          "locality" : "New York",
          "fullThroroughfare" : "",
          "label" : "",
          "streetAddress" : "",
          "stateCode" : "NY",
          "countryCode" : "US",
          "subAdministrativeArea" : "",
          "areaOfInterest" : [
            "Long Island"
          ],
          "administrativeArea" : "New York",
          "formattedAddressLines" : [
            "",
            "",
            ""
          ],
          "streetName" : "",
          "mapItemFullAddress" : ""
        },
        "location" : {
          "locationFinished" : true,
          "verticalAccuracy" : 80,
          "timeStamp" : 1705901792663,
          "altitude" : 0,
          "isInaccurate" : false,
          "horizontalAccuracy" : 80,
          "latitude" : ##.###############,
          "positionType" : "safeLocation",
          "isOld" : true,
          "longitude" : ##.###############,
          "floorLevel" : -1
        },
        "approvalState" : 1,
        "type" : 1,
        "identifier" : "########-####-####-####-############"
      }
    ],
    "groupIdentifier" : null,
    "location" : {
      "longitude" : ##.###############,
      "latitude" : ##.###############,
      "locationFinished" : true,
      "positionType" : "cached",
      "isOld" : true,
      "floorLevel" : -1,
      "altitude" : -1,
      "isInaccurate" : false,
      "horizontalAccuracy" : 35,
      "timeStamp" : 1705901798985,
      "verticalAccuracy" : -1
    },
    "crowdSourcedLocation" : {
      "floorLevel" : -1,
      "isOld" : true,
      "altitude" : -1,
      "verticalAccuracy" : -1,
      "timeStamp" : 1705901798985,
      "latitude" : ##.###############,
      "positionType" : "cached",
      "horizontalAccuracy" : 35,
      "locationFinished" : true,
      "longitude" : ##.###############,
      "isInaccurate" : false
    },
    "lostModeMetadata" : null,
    "identifier" : "########-####-####-####-############",
    "isFirmwareUpdateMandatory" : false,
    "productIdentifier" : "########-####-####-####-############",
    "systemVersion" : "2.0.61",
    "owner" : "owner@localhost",
    "name" : "Ernie",
    "productType" : {
      "productInformation" : {
        "manufacturerName" : "Apple",
        "modelName" : "",
        "productIdentifier" : 21760,
        "antennaPower" : 4,
        "vendorIdentifier" : 76
      },
      "type" : "b389"
    }
  },
{AirTag Object 2},
.
.
.
{AirTag Object n}]
```
</details>

The Bash script now finds the values for the location (TODO: add the crowd-sourced location) and add them to the CSV... the script below uses `jq` to properly parse the JSON. 

> [!WARNING]
> Works in Terminal, Isn't working from App (Automator)

> [!IMPORTANT]
> 
> ### Bash Script
> ```
> #!/bin/bash
> 
> csv_file="/Users/<username>/Documents/AirtagHistory/Airtag-Ernie.csv"
> items_data="/System/Volumes/Data/Users/<username>/Library/Caches/com.apple.findmy.fmipcore/Items.data"
> jq_path="/opt/homebrew/bin/jq"
> 
> if [ ! -f "$csv_file" ]; then
>  echo "Date,Latitude,Longitude,TimeStamp" > "$csv_file"
> fi
> 
> item=$("$jq_path" -c '.[] | select(.name == "Ernie")' $items_data)
> 
> echo "$(date),$(
>   echo "$item" | "$jq_path" -r '.location.latitude'
> ),$(
>   echo "$item" | "$jq_path" -r '.location.longitude'
> ),$(
>   echo "$item" | "$jq_path" -r '.location.timeStamp'
> )" >> "$csv_file"
> 
> echo "$(date),$(
>   echo "$item" | "$jq_path" -r '.crowdSourcedLocation.latitude'
> ),$(
>   echo "$item" | "$jq_path" -r '.crowdSourcedLocation.longitude'
> ),$(
>   echo "$item" | "$jq_path" -r '.crowdSourcedLocation.timeStamp'
> )" >> "$csv_file"
> ```
