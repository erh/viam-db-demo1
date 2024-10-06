<script lang="ts">
 import '../output.css'
 import '@viamrobotics/prime-core/prime.css';
 import { onMount } from 'svelte';
 import { Icon as PrimeIcon } from '@viamrobotics/prime-core';
 
 import { Logger } from "tslog";

 import {Coordinate} from "tsgeo/Coordinate";
 
 import { BSON } from "bsonfy";

 import {
   LinkedChart,
   LinkedLabel,
   LinkedValue
 } from "svelte-tiny-linked-charts"

 import * as VIAM from '@viamrobotics/sdk';

 import { base } from '$app/paths';
 
 const globalLogger = new Logger({ name: "global" });
 let globalClient: VIAM.RobotClient;
 let globalClientLastReset = new Date();
 let globalClientCloudMetaData = null;

 let globalCloudClient: VIAM.ViamClient;
 
 let globalData = {
   pos : new Coordinate(0,0),
   posString : "",
   speed : 0.0,

   cameraNames : [],

   gauges : {},
   gaugesToHistorical : {},
   
   numUpdates: 0,
   status: "Not connected yet",
   lastData: new Date(),   
 };

 var globalConfig = {
   movementSensorName : "",
   movementSensorProps : {},

   batterySensorName : ""
 };
 
 function gotNewData() {
   globalData.lastData = new Date();
 }
 
 function errorHandler(e) {
   globalLogger.error(e);
   var s = e.toString();
   globalData.status = "Error: " + s;

   var reset = false;

   var diff = new Date() - globalData.lastData;

   if (diff > 1000 * 30) {
     reset = true;
   }
   
   if (s.indexOf("SESSION_EXPIRED") >= 0) {
     reset = true;
   }

   if (reset && (new Date() - globalClientLastReset) > 1000 * 30) {
     globalLogger.warn("Forcing reconnect b/c session_expired");
     globalData.status = "Forcing reconnect b/c of error: " + e.toString();
     globalClient = null;
     globalClientLastReset = new Date();
   }

 }

 
 function doUpdate(loopNumber: int, client: VIAM.RobotClient){
   if (globalConfig.movementSensorName != "") {
     const msClient = new VIAM.MovementSensorClient(client, globalConfig.movementSensorName);
     
     msClient.getPosition().then((p) => {
       gotNewData();

       var myPos = new Coordinate(p.coordinate.latitude, p.coordinate.longitude);
       globalData.pos = myPos;
       globalData.posString = p.coordinate.latitude + "\n" + p.coordinate.longitude;
       
     }).catch(errorHandler);

     msClient.getLinearVelocity().then((v) => {
       globalData.speed = v.y * 1.94384;
     }).catch(errorHandler);
     
     msClient.getCompassHeading().then((ch) => {
       globalData.heading = ch;
     }).catch(errorHandler);

   }

   if (globalConfig.batterySensorName != "") {
     new VIAM.SensorClient(client, globalConfig.batterySensorName).getReadings().then((d) => {
       globalData.gauges["battery"] = d.percentage * 100;
     }).catch( (e) => {
       globalConfig.windSensorName = "";
       errorHandler(e);
     });
   
   }
   
   if (loopNumber % 30 == 2 ) {
     
     globalData.allResources.forEach( (r) => {
       if (r.subtype != "sensor") {
         return;
       }
       if (r.name.indexOf("fuel-") < 0 && r.name.indexOf("freshwater") < 0) {
         return;
       }
       
       var sp = r.name.split(":");
       var name = sp[sp.length-1];

       new VIAM.SensorClient(client, r.name).getReadings().then((raw) => {
         globalData.gauges[name] = raw;
       });
     });

   }
   
   
 }

 function doCameraLoop(loopNumber: int, client: VIAM.RobotClient) {
   var start = new Date();
   filterResources(globalData.allResources, "component", "camera").forEach( (r) => {
     if (globalData.cameraNames.indexOf(r.name) < 0) {
       globalData.cameraNames.push(r.name);
       globalData.cameraNames.sort();
     }

     new VIAM.CameraClient(client, r.name).getImage().then(
       function(img){
         var i = document.getElementById(r.name);
         if (i) {
           i.src = URL.createObjectURL(new Blob([img]));
         }
         console.log(  "time to fetch img: " + ((new Date()) - start) + " ms");
     }).catch(errorHandler);
     
   });

 }

 // t - type
 // st - subtype
 // n - name regex
 function filterResources(resources, t, st, n) {
   var a = [];
   for (var r of resources) {
     if (t != "", r.type != t) {
       continue;
     }

     if (st != "", r.subtype != st) {
       continue;
     }

     if (n != null && !r.name.match(n) ) {
       continue;
     }

     a.push(r);
   }

   return a;
 }
 
 async function updateResources(client: VIAM.RobotClient) {
   var resources = await client.resourceNames();
   globalData.allResources = resources;

   await setupMovementSensor(client, resources);
   await setupBatterySensor(client, resources);

   console.log("globalConfig", globalConfig);

 }

 async function setupBatterySensor(client: VIAM.RobotClient, resources) {
   resources = filterResources(resources, "component", "sensor", /power-report/);
   
   for (var r of resources) {
     globalConfig.batterySensorName = r.name;
   }
   
 }

 
 async function setupMovementSensor(client: VIAM.RobotClient, resources) {
   resources = filterResources(resources, "component", "movement_sensor", null);
   
   // pick best movement sensor
   var bestName = "";
   var bestScore = 0;
   var bestProp = {};
   
   for (var r of resources) {
     const msClient = new VIAM.MovementSensorClient(client, r.name);
     var prop = await msClient.getProperties();

     var score = 0;
     if (prop.positionSupported) {
       score++;
     }
     if (prop.linearVelocitySupported) {
       score++;
     }
     if (prop.compassHeadingSupported) {
       score++;
     }

     if (score > bestScore) {
       bestName = r.name;
       bestScore = score;
       bestProp = prop;
     }
   }
   
   globalConfig.movementSensorName = bestName;
   globalConfig.movementSensorProps = bestProp;
 }
 
 async function updateAndLoop() {
   globalData.numUpdates++;
   
   if (!globalClient) {
     try {
       globalClient = await connect();
       await updateResources(globalClient);

     } catch(error) {
       globalData.status = "Connect failed: " + error;
       globalClient = null;
     }
   } else if (globalClient.numUpdates % 300 == 0) {
     await updateResources(globalClient);     
   }
   
   var client = globalClient;
   
   if (client) {
     doUpdate(globalData.numUpdates, client);
     doCameraLoop(globalData.numUpdates, client);
   }

   setTimeout(updateAndLoop, 1000);
   if (globalData.numUpdates == 1) {
     setTimeout(updateCloudDataAndLoop, 1000);
   }
 }

 async function updateCloudDataAndLoop() {
   const urlParams = new URLSearchParams(window.location.search);
   
   if (!globalCloudClient) {
     try {
       const opts: VIAM.ViamClientOptions = {
         credential: {
           type: 'api-key',
           authEntity: urlParams.get("authEntity"),
           payload: urlParams.get("api-key"),
         },
       };
       
       globalCloudClient = await VIAM.createViamClient(opts);
       
     } catch( error ) {
       console.log("cannot connect to cloud: " + error);
     }
   }
   
   if (globalCloudClient) {
     var dc = globalCloudClient.dataClient;

     try {
       await updateGaugeGraphs(globalCloudClient.dataClient);
     } catch(error) {
       console.log("error calling updateGaugeGraphs: " + error);
     }
   }

   setTimeout(updateCloudDataAndLoop, 1000);
 }

 async function getDataViaMQL(dc, g, startTime, key) {
   var match = {
     "location_id" : globalClientCloudMetaData.locationId,
     "robot_id" : globalClientCloudMetaData.machineId,
     "component_name" : g,
     time_received: { $gte: startTime }
   };
   
   var group = {
     "_id": { "$concat" : [
                                  { "$toString": { "$substr" : [ { "$year": "$time_received" } , 2, -1 ] } },
                                  "-",
                                  { "$toString" : { "$month": "$time_received" } },
                                  "-",
                                  { "$toString" : { "$dayOfMonth": "$time_received" } },
                                  " ",
                                  { "$toString" : { "$hour": "$time_received" } },
                                  ":",
                                  { "$toString" : { "$multiply" : [ 15, { "$floor" : { "$divide": [ { "$minute": "$time_received"}, 15] } } ] } }
                                  ] },
     "ts" : { "$min" : "$time_received" },
     "min" : { "$min" : "$data.readings." + key },
     "max" : { "$max" : "$data.readings." + key }
   };
   
   console.log(match);
   console.log(group);

   var query = [
     BSON.serialize( { "$match" : match } ),
     BSON.serialize( { "$group" : group } ),
     BSON.serialize( { "$sort" : { ts : -1 } } ),
     BSON.serialize( { "$limit" : (24 * 4) } ),
     BSON.serialize( { "$sort" : { ts : 1 } } ),
   ];

   try {
     var data = await dc.tabularDataByMQL(globalClientCloudMetaData.primaryOrgId, query);
     return data;
   } catch(error) {
     console.log(error);
     return [];
   }
 }

 
 async function updateGaugeGraphs(dc) {
   var startTime = new Date(new Date() - 86400 * 1000);

   for ( var g in globalData.gauges ) {
     var h = globalData.gaugesToHistorical[g];
     if (h && (new Date() - h.ts) < 60000) {
       continue;
     }

     var timeStart = new Date();
     var data = await getDataViaMQL(dc, g, startTime);
     var getDataTime = (new Date()).getTime() - timeStart.getTime();
     
     console.log("time to get graph data for " + g + " took " + getDataTime + " and had " + data.length + " points");
     
     h = { ts : new Date(), data : data };
     globalData.gaugesToHistorical[g] = h;
   }

 }
 
 async function connect(): VIAM.RobotClient {
   const urlParams = new URLSearchParams(window.location.search);
   
   const host = urlParams.get("host");
   const apiKey = urlParams.get("api-key");
   const authEntity = urlParams.get("authEntity");
   
   const credential = {
     type: 'api-key',
     payload: apiKey
   };

   var c = await VIAM.createRobotClient({
     host,
     credential: credential,
     authEntity: authEntity,
     signalingAddress: 'https://app.viam.com:443',

     // optional: configure reconnection options
     reconnectMaxAttempts: 20,
     reconnectMaxWait: 5000,
   });

   globalData.status = "Connected";
   
   globalLogger.info('connected!');
   
   c.on('disconnected', disconnected);
   c.on('reconnected', reconnected);

   globalClientCloudMetaData = await c.getCloudMetadata();
   console.log(globalClientCloudMetaData);
   return c;
 }

 async function disconnected(event) {
   globalData.status = "Disconnected";
   globalLogger.warn('The robot has been disconnected. Trying reconnect...');
 }

 async function reconnected(event) {
   globalData.status = "Connected";
   globalLogger.warn('The robot has been reconnected. Work can be continued.');
 }


 async function start() {

   try {
     updateAndLoop();
     return {};
   } catch (error) {
     errorHandler(error);
   }
 }

 onMount(start);

 function gaugesToArray(gauges) {
   var names = Object.keys(gauges);
   names.sort();

   var a = [];
   
   for ( var i = 0; i < names.length; i++) {
     var n = names[i];
     a.push( [ n, gauges[n] ]);
   }
   return a;
 }

 function gauageHistoricalToLinkedChart(data) {
   var res = {};
   for (var d in data.data) {
     var dd = data.data[d];
     res[dd._id] = dd.min;
   }
   return res;
 }
</script>


<main class="w-dvw lg:h-dvh p-2 grid grid-cols-1 lg:grid-cols-4 grid-rows-3 lg:grid-rows-6 gap-2">

  <div class="lg:col-span-3 border border-light bg-white">
    <table>
      <tr>
        <th></th>
        <th>Left</th>
        <th>Right</th>
      </tr>
      <tr>
        <th>Front</th>
        <td><img id="front-left" alt="front-left" /></td>
        <td><img id="front-right" alt="front-right" /></td>
      </tr>
      <tr>
        <th>Port</th>
        <td><img id="port-left" alt="port-left" /></td>
        <td><img id="port-right" alt="port-right" /></td>
      </tr>
      <tr>
        <th>Starboard</th>
        <td><img id="starboard-left" alt="starboard-left" /></td>
        <td><img id="starboard-right" alt="starboard-right" /></td>
      </tr>
      <tr>
        <th>Read</th>
        <td><img id="rear-left" alt="rear-left" /></td>
        <td><img id="rear-right" alt="rear-right" /></td>
      </tr>

    </table>
  </div>
    
  <aside class="lg:row-span-6 flex flex-col gap-4 border border-light p-1 bg-white min-h-full">
    {#if globalData.status === "Connected"}
      <div class="flex gap-2 items-center w-full min-h-6 px-2 border border-success-medium bg-success-light">
        <PrimeIcon
          name="broadcast"
          cx="text-success-dark"
        />
        <div class="text-sm text-success-dark">{globalData.status}</div>
      </div>
    {:else}
    <div class="flex gap-2 items-center w-full min-h-6 px-2 border border-info-medium bg-info-light">
      <PrimeIcon
        name="broadcast-off"
        cx="text-info-dark"
      />
      <div class="text-sm text-info-dark">{globalData.status}</div>
    </div>
    {/if}

    <div id="navData" class="flex flex-col divide-y">
      {#if globalConfig.movementSensorProps.linearVelocitySupported}
        <div class="flex gap-2 p-2 text-lg">
          <div class="min-w-32">Speed<br></div>
          <div>
            <span class="font-bold">{globalData.speed.toFixed(2)}</span>
          </div>
        </div>
      {/if}
      {#if globalConfig.movementSensorProps.positionSupported}
        <div class="flex gap-2 p-2 text-lg">
          <div class="min-w-32">Location</div>
          <span class="font-bold" style="white-space: pre-line;">{globalData.posString}</span>
        </div>
      {/if}
      {#if globalConfig.movementSensorProps.compassHeadingSupported}
        <div class="flex gap-2 p-2 text-lg">
          <div class="min-w-32">Heading</div>
          <div>
            <span class="font-bold">{@html globalData.heading}</span>
          </div>
        </div>
      {/if}
      <div class="flex flex-col divide-y">
        {#each gaugesToArray(globalData.gauges) as [key, value]}
          <section class="overflow-visible flex gap-2 p-2 text-lg">
            <h2 class="min-w-32 capitalize">{key}</h2>
            <div class="grow">
              <div class="flex gap-1 font-bold">
                <div>{value.toFixed(0)} %</div>
              </div>
              {#if globalData.gaugesToHistorical[key]}
              <div class="relative">
                <div role="article" tabindex="-1" class="peer bg-light hover:cursor-pointer">
                  <LinkedChart
                    data={gauageHistoricalToLinkedChart(globalData.gaugesToHistorical[key])}
                    style="width: 100%;"
                    width="100"
                    type="line"
                    scaleMax=100
                    linked="{key}"
                    uid="{key}"
                    barMinWidth="1"
                    grow
                  />
                </div>
                <div
                  class="hidden peer-hover:block z-10 text-nowrap -bottom-8 right-1 absolute border border-medium bg-white px-2 w-fit"
                >
                  <LinkedValue uid="{key}" />
                  <LinkedLabel linked="{key}"/>
                </div>
              </div>
            {/if}   
            </div>
          </section>
        {/each}
      </div>
    </div>

    <div class="grow text-xs flex flex-col flex-col-reverse text-gray-500 text-right">{globalData.numUpdates}</div>
  </aside>

</main>
