<script lang="ts">

import { onMount, onDestroy } from 'svelte';
import { grpc } from '@improbable-eng/grpc-web';
import { Client, movementSensorApi as movementsensorApi, type ServiceError } from '@viamrobotics/sdk';
import type { ResponseStream, commonApi, robotApi } from '@viamrobotics/sdk';
import { displayError } from '@/lib/error';
import { rcLogConditionally } from '@/lib/log';

export let name: string
export let client: Client
export let statusStream: ResponseStream<robotApi.StreamStatusResponse> | null

let orientation: commonApi.Orientation.AsObject | undefined;
let angularVelocity: commonApi.Vector3.AsObject | undefined;
let linearVelocity: commonApi.Vector3.AsObject | undefined;
let linearAcceleration: commonApi.Vector3.AsObject | undefined;
let compassHeading: number | undefined;
let coordinate: commonApi.GeoPoint.AsObject | undefined;
let altitudeM: number | undefined;
let properties: movementsensorApi.GetPropertiesResponse.AsObject | undefined;

let refreshId = -1;

const refresh = async () => {
  properties = await new Promise((resolve) => {
    const req = new movementsensorApi.GetPropertiesRequest();
    req.setName(name);

    rcLogConditionally(req);
    client.movementSensorService.getProperties(
      req,
      new grpc.Metadata(),
      (err: ServiceError | null, resp: movementsensorApi.GetPropertiesResponse | null) => {
        if (err) {
          if (err.message === 'Response closed without headers') {
            refreshId = window.setTimeout(refresh, 500);
            return;
          }
          return displayError(err);
        }

        resolve(resp!.toObject());
      }
    );
  });

  if (properties?.orientationSupported) {
    const req = new movementsensorApi.GetOrientationRequest();
    req.setName(name);

    rcLogConditionally(req);
    client.movementSensorService.getOrientation(
      req,
      new grpc.Metadata(),
      (err: ServiceError | null, resp: movementsensorApi.GetOrientationResponse | null) => {
        if (err) {
          return displayError(err);
        }

        orientation = resp!.toObject().orientation;
      }
    );
  }

  if (properties?.angularVelocitySupported) {
    const req = new movementsensorApi.GetAngularVelocityRequest();
    req.setName(name);

    rcLogConditionally(req);
    client.movementSensorService.getAngularVelocity(
      req,
      new grpc.Metadata(),
      (err: ServiceError | null, resp: movementsensorApi.GetAngularVelocityResponse | null) => {
        if (err) {
          return displayError(err);
        }

        angularVelocity = resp!.toObject().angularVelocity;
      }
    );
  }

  if (properties?.linearAccelerationSupported) {
    const req = new movementsensorApi.GetLinearAccelerationRequest();
    req.setName(name);

    rcLogConditionally(req);
    client.movementSensorService.getLinearAcceleration(
      req,
      new grpc.Metadata(),
      (err: ServiceError | null, resp: movementsensorApi.GetLinearAccelerationResponse | null) => {
        if (err) {
          return displayError(err);
        }

        linearAcceleration = resp!.toObject().linearAcceleration;
      }
    );
  }

  if (properties?.linearVelocitySupported) {
    const req = new movementsensorApi.GetLinearVelocityRequest();
    req.setName(name);

    rcLogConditionally(req);
    client.movementSensorService.getLinearVelocity(
      req,
      new grpc.Metadata(),
      (err: ServiceError | null, resp: movementsensorApi.GetLinearVelocityResponse | null) => {
        if (err) {
          return displayError(err);
        }

        linearVelocity = resp!.toObject().linearVelocity;
      }
    );
  }

  if (properties?.compassHeadingSupported) {
    const req = new movementsensorApi.GetCompassHeadingRequest();
    req.setName(name);

    rcLogConditionally(req);
    client.movementSensorService.getCompassHeading(
      req,
      new grpc.Metadata(),
      (err: ServiceError | null, resp: movementsensorApi.GetCompassHeadingResponse | null) => {
        if (err) {
          return displayError(err);
        }

        compassHeading = resp!.toObject().value;
      }
    );
  }

  if (properties?.positionSupported) {
    const req = new movementsensorApi.GetPositionRequest();
    req.setName(name);

    rcLogConditionally(req);
    client.movementSensorService.getPosition(
      req,
      new grpc.Metadata(),
      (err: ServiceError | null, resp: movementsensorApi.GetPositionResponse | null) => {
        if (err) {
          return displayError(err);
        }

        const temp = resp!.toObject();
        coordinate = temp.coordinate;
        // eslint-disable-next-line @typescript-eslint/ban-ts-comment
        // @ts-ignore `altitudeM` is correct from teh protos
        altitudeM = temp.altitudeM;
      }
    );
  }

  refreshId = window.setTimeout(refresh, 500);
  statusStream?.on('end', () => clearTimeout(refreshId));
};

onMount(() => {
  refreshId = window.setTimeout(refresh, 500);
});

onDestroy(() => {
  clearTimeout(refreshId);
});

</script>

<v-collapse title={name}>
  <v-breadcrumbs slot="title" crumbs="movement_sensor" />
  <div class="flex flex-wrap gap-4 text-sm border border-t-0 border-medium p-4">
    {#if properties?.positionSupported}
      <div class="overflow-auto">
        <h3 class="mb-1">Position</h3>
        <table class="w-full border border-t-0 border-medium p-4">
          <tr>
            <th class="border border-medium p-2">
              Latitude
            </th>
            <td class="border border-medium p-2">
              {coordinate?.latitude.toFixed(6)}
            </td>
          </tr>
          <tr>
            <th class="border border-medium p-2">
              Longitude
            </th>
            <td class="border border-medium p-2">
              {coordinate?.longitude.toFixed(6)}
            </td>
          </tr>
          <tr>
            <th class="border border-medium p-2">
              Altitide (m)
            </th>
            <td class="border border-medium p-2">
              {altitudeM?.toFixed(2)}
            </td>
          </tr>
        </table>
        <a
          class="text-[#045681] underline"
          href={`https://www.google.com/maps/search/${coordinate?.latitude},${coordinate?.longitude}`}
        >
          google maps
        </a>
      </div>
    {/if}

    {#if properties?.orientationSupported}
      <div class="overflow-auto">
        <h3 class="mb-1">
          Orientation (degrees)
        </h3>
        <table class="w-full border border-t-0 border-medium p-4">
          <tr>
            <th class="border border-medium p-2">
              OX
            </th>
            <td class="border border-medium p-2">
              {orientation?.oX.toFixed(2)}
            </td>
          </tr>
          <tr>
            <th class="border border-medium p-2">
              OY
            </th>
            <td class="border border-medium p-2">
              {orientation?.oY.toFixed(2)}
            </td>
          </tr>
          <tr>
            <th class="border border-medium p-2">
              OZ
            </th>
            <td class="border border-medium p-2">
              {orientation?.oZ.toFixed(2)}
            </td>
          </tr>
          <tr>
            <th class="border border-medium p-2">
              Theta
            </th>
            <td class="border border-medium p-2">
              {orientation?.theta.toFixed(2)}
            </td>
          </tr>
        </table>
      </div>
    {/if}

    {#if properties?.angularVelocitySupported}
      <div class="overflow-auto">
        <h3 class="mb-1">
          Angular Velocity (degrees/second)
        </h3>
        <table class="w-full border border-t-0 border-medium p-4">
          <tr>
            <th class="border border-medium p-2">
              X
            </th>
            <td class="border border-medium p-2">
              {angularVelocity?.x.toFixed(2)}
            </td>
          </tr>
          <tr>
            <th class="border border-medium p-2">
              Y
            </th>
            <td class="border border-medium p-2">
              {angularVelocity?.y.toFixed(2)}
            </td>
          </tr>
          <tr>
            <th class="border border-medium p-2">
              Z
            </th>
            <td class="border border-medium p-2">
              {angularVelocity?.z.toFixed(2)}
            </td>
          </tr>
        </table>
      </div>
    {/if}

    {#if properties?.linearVelocitySupported}
      <div class="overflow-auto">
        <h3 class="mb-1">
          Linear Velocity (m/s)
        </h3>
        <table class="w-full border border-t-0 border-medium p-4">
          <tr>
            <th class="border border-medium p-2">
              X
            </th>
            <td class="border border-medium p-2">
              {linearVelocity?.x.toFixed(2)}
            </td>
          </tr>
          <tr>
            <th class="border border-medium p-2">
              Y
            </th>
            <td class="border border-medium p-2">
              {linearVelocity?.y.toFixed(2)}
            </td>
          </tr>
          <tr>
            <th class="border border-medium p-2">
              Z
            </th>
            <td class="border border-medium p-2">
              {linearVelocity?.z.toFixed(2)}
            </td>
          </tr>
        </table>
      </div>
    {/if}

    {#if properties?.linearAccelerationSupported}
      <div class="overflow-auto">
        <h3 class="mb-1">
          Linear Acceleration (m/second^2)
        </h3>
        <table class="w-full border border-t-0 border-medium p-4">
          <tr>
            <th class="border border-medium p-2">
              X
            </th>
            <td class="border border-medium p-2">
              {linearAcceleration?.x.toFixed(2)}
            </td>
          </tr>
          <tr>
            <th class="border border-medium p-2">
              Y
            </th>
            <td class="border border-medium p-2">
              {linearAcceleration?.y.toFixed(2)}
            </td>
          </tr>
          <tr>
            <th class="border border-medium p-2">
              Z
            </th>
            <td class="border border-medium p-2">
              {linearAcceleration?.z.toFixed(2)}
            </td>
          </tr>
        </table>
      </div>
    {/if}

    {#if properties?.compassHeadingSupported}
      <div class="overflow-auto">
        <h3 class="mb-1">
          Compass Heading
        </h3>
        <table class="w-full border border-t-0 border-medium p-4">
          <tr>
            <th class="border border-medium p-2">
              Compass
            </th>
            <td class="border border-medium p-2">
              {compassHeading?.toFixed(2)}
            </td>
          </tr>
        </table>
      </div>
    {/if}
  </div>
</v-collapse>