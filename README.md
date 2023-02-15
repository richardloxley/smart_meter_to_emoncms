# Smart meter to EmonCMS

## Background

[EmonCMS](https://github.com/emoncms/emoncms) is a web application that can log and visualise energy usage.  It is part of the [Open Energy Monitor project](https://openenergymonitor.org/).

Users commonly use devices that can measure electricity usage via a current clamp, and upload it to EmonCMS.

However gas usage is harder to record, and monitoring electricity use requires these current clamp devices in the home.

In the UK, smart meters are being rolled out.  They can record electricity and gas usage with a 30 minute granularity.  The smart meter service allows the home occupier to share this data with third party services.  One such third-party [n3rgy](https://data.n3rgy.com/consumer/home) offers a free service to consumers to access this data.

This project uses n3rgy's API to download the electricity and gas consumption data, and then upload it to an EmonCMS server.

It also uses a public API from National Grid to convert gas volume to kWh using the published calorific value of the gas in your area for a given day.

## Pre-requisites

- A UK smart meter with an "In Home Device" (IHD) - the table-top display that shows your energy usage
- A free account with n3rgy (see below for how to sign up)
- An account on an EmonCMS server 
- A computer that can run PHP code

## Sign up for n3rgy

First look at your **In Home Device** (the table-top display that shows your energy usage).  On the back (or inside any battery compartment) it should have a **MAC address** (a 12 digit hexadecimal number).  You will need this to prove you are the home occupier.

Then you need either the **MPAN number** from your electricity meter *or* your **MPRN number** from your gas meter.  You can find this:

- On a bill from your energy supplier, or
- Within the menus on the In Home Device (possibly under "Meters"), or
- Within the menus on the smart meter (look up the model number of the meter online to find instructions on how to find it)

The go to the [n3rgy sign-up page](https://data.n3rgy.com/consumer/sign-up).

- Enter your MPAN or MPRN
- Accept the terms
- Fill in when you moved in to the property
- Enter the MAC address of your IHD

n3rgy will start indexing the data from your meter.  It might take a little while until the data is available via the API.

If you just want to browse the data manually, you can [download CSV files here](https://data.n3rgy.com/consumer/download-data)

## Set up the script

Download *smart_meter_to_emoncms*.  It's a PHP script, so you will need PHP installed on the computer you run it on.

First edit the file, to set up the configuration variables at the top:

### EMONCMS_URL ###

The URL of your EmonCMS server, e.g. https://www.example.com/emoncms/

### EMONCMS_KEY ###

The "Read & Write API Key" for your EmonCMS account.  You can find it under "My Account" on your EmonCMS server.

### EMONCMS_ELECTRICITY_INPUT ###

The name of the input in EmonCMS for the electricity imported (in watts), e.g. "electricity", "import", "meter_import", etc.  Leave blank if you don't want to import electricity data.

If it doesn't yet exist it will be created by the script.  If you want to import historical data it's best to let it be created by the script, as the script will only import data later than the most recent reading (if any) in EmonCMS.

### EMONCMS_GAS_INPUT ###

The name of the input in EmonCMS for the gas used (in watts), e.g. "gas", "gas_meter", etc.  Leave blank if you don't want to import gas data.

If it doesn't yet exist it will be created by the script.  If you want to import historical data it's best to let it be created by the script, as the script will only import data later than the most recent reading (if any) in EmonCMS.

### MAC_ADDRESS ###

The MAC address of your IHD (In Home Display), which is used to authenticate with n3rgy - the same as you used when you signed up to n3rgy.

### GAS_REGION ###

The code for your gas region (used to calculate the local calorific value).  If left blank the script will use the UK average of 39.5 but the kWh values will be less accurate.  (You don't need to set this if you are only importing electricity.)

To find your region, you can [look it up here](https://www.energybrokers.co.uk/gas/gas-network).

Then use this list to find the code for your region:        

```
PUBOB4507       LDZ(EA)         East Anglia
PUBOB4508       LDZ(EM)         East Midlands
PUBOB4510       LDZ(NE)         North East
PUBOB4509       LDZ(NO)         Northern
PUBOB4511       LDZ(NT)         North Thames
PUBOB4512       LDZ(NW)         North West
PUBOB4513       LDZ(SC)         Scotland
PUBOB4514       LDZ(SE)         South East
PUBOB4515       LDZ(SO)         Southern
PUBOB4516       LDZ(SW)         South West
PUBOB4517       LDZ(WM)         West Midlands
PUBOB4518       LDZ(WN)         Wales North
PUBOB4519       LDZ(WS)         Wales South
PUBOB4521       Oban
PUBOB4520       Stornoway
PUBOB4522       Stranraer
PUBOBJ1661      Thurso
PUBOBJ1662      Wick
PUBOBJ1660      Campbeltown
```

e.g. if you're in the LDZ SW region, set GAS_REGION to "PUBOB4516".

### DEBUG ###

Set this to "true" if you want the readings to be output to the screen/terminal when the script runs (as well as sending them to EmonCMS).  Set it to "false" to silently upload the readings.  Error messages will always be output.

### Other variables ###

There are some other global variables at the top of the file:

- VOLUME_CORRECTION_FACTOR
- DEFAULT_CALORIFIC_VALUE
- ELECTRICITY_CONSUMPTION_URL
- GAS_CONSUMPTION_URL
- CALORIFIC_VALUE_URL

You shouldn't need to change these unless the APIs change, or the way gas is billed in the UK changes.

## Run the script for the first time to set up the inputs

You may need to set the script to be executable first:

```
chmod +x smart_meter_to_emoncms
```

Then you can run it:

```
./smart_meter_to_emoncms
```

Or you can force it to run by passing it to the PHP command:

```
php smart_meter_to_emoncms
```

If the EmonCMS inputs haven't yet been created, then first time you run the script, it will create them, and give a message such as:

```
Created EmonCMS input "electricity".
**** IMPORTANT **** create any feeds that depend on this input before running this script again.
Created EmonCMS input "gas".
**** IMPORTANT **** create any feeds that depend on this input before running this script again.
```

**NB. You should now set up the EmonCMS feeds (next section) before you run the script again!**

## Set up the EmonCMS feeds

- Log into your EmonCMS account.
<br/>

- Go to "Inputs".
- Find the input(s) that have just been created.
- Click the "spanner" icon on the right of each input.
- Add a process "Log to feed", with a fixed interval of 30 minutes.  You can use the default name (the same as the input).
- (Optionally) add another process "Power to kWh", with a fixed interval of 30 minutes.  Append "_kwh" to the default name.
- Click "Changed, press to save".
- Click "Close".
- Repeat for the other input (if you are importing both electricity and gas).
<br/>

- Go to "Feeds".
- Find the feed(s) that were named the same as the inputs.
- Tick the box(es) next to them.
- Click the "pencil" (edit) button at the top of the list.
- Change the Feed Unit to "Watt (W)".
- Click "Save".

## Run the script for the second time to import historical data

Now run the script again:

```
./smart_meter_to_emoncms
```

or:

```
php smart_meter_to_emoncms
```

It should download any historical data from the last 90 days and upload it to EmonCMS.  It might take a minute or two, particularly if you're downloading gas data as it has to fetch the calorific value data from National Grid, which is quite slow.

## (Ongoing) run the script to fetch new data

Then whenever you wish to check for new data, run the script again:

```
./smart_meter_to_emoncms
```

or:

```
php smart_meter_to_emoncms
```

This should be relatively quick, as it only downloads data since the last import.

I believe the smart meter data is only updated once a day, so you can run the script manually when you want to import new data.  Make sure this is at least once every 90 days, as n3rgy have a limit of 90 days on downloading data.

Or you could set it up as a cron job on a suitable machine to keep the data up-to-date automatically.

I have put the script in /etc/cron.hourly/ on my Linux machine, so the data is uploaded promptly soon after it becomes available.  The script does nothing if no new data is available.  You may want to set DEBUG to "false" if you are running it as a cron job.

## Further reading

### n3rgy API

The API is quite simple, here are some examples using curl:

```
Consumption:
    curl -H "Authorization: MACADDRESS" https://consumer-api.data.n3rgy.com/electricity/consumption/1
    curl -H "Authorization: MACADDRESS" https://consumer-api.data.n3rgy.com/gas/consumption/1
    
Date ranges:
    curl -H "Authorization: MACADDRESS" https://consumer-api.data.n3rgy.com/gas/consumption/1?start=202301260000\&end=202301262330

Tariff:
    curl -H "Authorization: MACADDRESS" https://consumer-api.data.n3rgy.com/electricity/tariff/1
    
Export meters (untested):
    curl -H "Authorization: MACADDRESS" https://consumer-api.data.n3rgy.com/electricity/production/1
```

### Other n3rgy client implementations

- https://github.com/n3rgy/data
- https://github.com/jonathanadams/n3rgy-client

### More information about third-party smart meter data access

- https://guylipman.medium.com/letting-people-access-their-electricity-data-e3d36ad9b6c0
- https://www.smartme.co.uk/meter-data.html

### National Grid calorific values

You can download calorific values using the [National Grid Data Item Explorer](https://gasdata.nationalgrid.com/DataItemExplorer).

There is some very limited data (including acceptable use) on the API here: https://www.nationalgas.com/sites/gas/files/documents/8589935564-API%20Guidance%20v1.0%2020.6.16.pdf

I discovered more by reading the code of the Data Item Explorer website, including the gas area codes.

Here are some examples of using the API with curl:

```
CSV
curl "https://gasdata.nationalgrid.com/DataItemViewer/DownloadFile?LatestValue=true&Applicable=applicableFor&FromUtcDatetime=2023-02-05T00:00:00.000Z&ToUtcDateTime=2023-02-05T00:00:00.000Z&PublicationObjectStagingIds=PUBOB4516&FileType=Csv"

XML
curl "https://gasdata.nationalgrid.com/DataItemViewer/DownloadFile?LatestValue=true&Applicable=applicableFor&FromUtcDatetime=2023-02-05T00:00:00.000Z&ToUtcDateTime=2023-02-05T00:00:00.000Z&PublicationObjectStagingIds=PUBOB4516&FileType=Xml"
```
