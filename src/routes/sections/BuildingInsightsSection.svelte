<!--
 Copyright 2023 Google LLC

 Licensed under the Apache License, Version 2.0 (the "License");
 you may not use this file except in compliance with the License.
 You may obtain a copy of the License at

      https://www.apache.org/licenses/LICENSE-2.0

 Unless required by applicable law or agreed to in writing, software
 distributed under the License is distributed on an "AS IS" BASIS,
 WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 See the License for the specific language governing permissions and
 limitations under the License.
 -->

<script lang="ts">
  /* global google */

  import type { MdDialog } from '@material/web/dialog/dialog';
  import Expandable from '../components/Expandable.svelte';
  import {
    type BuildingInsightsResponse,
    type RequestError,
    findClosestBuilding,
    type SolarPanelConfig,
  } from '../solar';
  import Show from '../components/Show.svelte';
  import SummaryCard from '../components/SummaryCard.svelte';
  import { createPalette, normalize, rgbToColor } from '../visualize';
  import { panelsPalette } from '../colors';
  import InputBool from '../components/InputBool.svelte';
  import InputPanelsCount from '../components/InputPanelsCount.svelte';
  import { showNumber } from '../utils';
  import NumberInput from '../components/InputNumber.svelte';
  import Gauge from '../components/Gauge.svelte';

  export let expandedSection: string;
  export let buildingInsights: BuildingInsightsResponse | undefined;
  export let configId: number | undefined;
  export let panelCapacityWatts: number;
  export let showPanels: boolean;

  export let googleMapsApiKey: string;
  export let geometryLibrary: google.maps.GeometryLibrary;
  export let location: google.maps.LatLng;
  export let map: google.maps.Map;

  const icon = 'home';
  const title = 'Building Insights endpoint';

  let requestSent = false;
  let requestError: RequestError | undefined;
  let apiResponseDialog: MdDialog;

  // Add new controls
  let showSegments = true;
  let segmentFilter = "";
  let disableAnimation = true;
  let showSolarPotentialLayer = true; // Always true, no longer toggleable or exported

  // Make each toggle directly update the visibility
  function updatePanelVisibility() {
    if (solarPanels.length === 0) return;
    
    solarPanels.forEach((panel, i) => {
      if (disableAnimation) {
        panel.setMap(showPanels ? map : null);
      } else {
        panel.setMap(showPanels && panelConfig && i < panelConfig.panelsCount ? map : null);
      }
    });
  }
  
  function updateSegmentVisibility() {
    if (roofSegmentPolygons.length === 0) return;
    
    const segmentNumbers = segmentFilter.trim() === "" ? 
      null : 
      segmentFilter.split(',').map(s => parseInt(s.trim())).filter(n => !isNaN(n));
    
    roofSegmentPolygons.forEach((segment) => {
      // @ts-ignore - Using custom property
      const segmentIndex = segment.segmentIndex;
      const shouldShow = showSegments && 
        (segmentNumbers === null || segmentNumbers.includes(segmentIndex));
      segment.setMap(shouldShow ? map : null);
    });
    
    roofSegmentLabels.forEach((label) => {
      // @ts-ignore - Using custom property
      const segmentIndex = label.segmentIndex;
      const shouldShow = showSegments && 
        (segmentNumbers === null || segmentNumbers.includes(segmentIndex));
      label.setMap(shouldShow ? map : null);
    });
  }
  
  // React to toggle changes
  $: if (showPanels !== undefined) {
    // Make sure we update UI after toggle change is complete
    setTimeout(() => updatePanelVisibility(), 0);
  }
  
  $: if (disableAnimation !== undefined) {
    // Make sure we update UI after toggle change is complete
    setTimeout(() => updatePanelVisibility(), 0);
  }
  
  $: if (showSegments !== undefined) {
    // Make sure we update UI after toggle change is complete
    setTimeout(() => updateSegmentVisibility(), 0);
  }
  
  $: if (segmentFilter !== undefined) {
    // Make sure we update UI after toggle change is complete
    setTimeout(() => updateSegmentVisibility(), 0);
  }

  let panelConfig: SolarPanelConfig | undefined;
  $: if (buildingInsights && configId !== undefined) {
    panelConfig = buildingInsights.solarPotential.solarPanelConfigs[configId];
  }

  let solarPanels: google.maps.Polygon[] = [];
  $: if (solarPanels.length > 0) {
    // Use our updatePanelVisibility helper to handle all panel visibility changes
    updatePanelVisibility();
  }

  // Add variables for roof segments visualization
  let roofSegmentPolygons: google.maps.Polygon[] = [];
  let roofSegmentLabels: google.maps.Marker[] = [];
  
  let panelCapacityRatio = 1.0;
  $: if (buildingInsights) {
    const defaultPanelCapacity = buildingInsights.solarPotential.panelCapacityWatts;
    panelCapacityRatio = panelCapacityWatts / defaultPanelCapacity;
  }

  export async function showSolarPotential(location: google.maps.LatLng) {
    if (requestSent) {
      return;
    }

    console.log('showSolarPotential');
    buildingInsights = undefined;
    requestError = undefined;

    // Clear existing visualizations
    solarPanels.map((panel) => panel.setMap(null));
    solarPanels = [];
    
    // Clear existing roof segment visualizations
    roofSegmentPolygons.map((polygon) => polygon.setMap(null));
    roofSegmentPolygons = [];
    roofSegmentLabels.map((marker) => marker.setMap(null));
    roofSegmentLabels = [];

    requestSent = true;
    try {
      buildingInsights = await findClosestBuilding(location, googleMapsApiKey);
    } catch (e) {
      requestError = e as RequestError;
      return;
    } finally {
      requestSent = false;
    }

    // Create the solar panels on the map.
    const solarPotential = buildingInsights.solarPotential;
    const palette = createPalette(panelsPalette).map(rgbToColor);
    const minEnergy = solarPotential.solarPanels.slice(-1)[0].yearlyEnergyDcKwh;
    const maxEnergy = solarPotential.solarPanels[0].yearlyEnergyDcKwh;
    solarPanels = solarPotential.solarPanels.map((panel) => {
      const [w, h] = [solarPotential.panelWidthMeters / 2, solarPotential.panelHeightMeters / 2];
      const points = [
        { x: +w, y: +h }, // top right
        { x: +w, y: -h }, // bottom right
        { x: -w, y: -h }, // bottom left
        { x: -w, y: +h }, // top left
        { x: +w, y: +h }, //  top right
      ];
      const orientation = panel.orientation == 'PORTRAIT' ? 90 : 0;
      const azimuth = solarPotential.roofSegmentStats[panel.segmentIndex].azimuthDegrees;
      const colorIndex = Math.round(normalize(panel.yearlyEnergyDcKwh, maxEnergy, minEnergy) * 255);
      return new google.maps.Polygon({
        paths: points.map(({ x, y }) =>
          geometryLibrary.spherical.computeOffset(
            { lat: panel.center.latitude, lng: panel.center.longitude },
            Math.sqrt(x * x + y * y),
            Math.atan2(y, x) * (180 / Math.PI) + orientation + azimuth,
          ),
        ),
        strokeColor: '#B0BEC5',
        strokeOpacity: 0.9,
        strokeWeight: 1,
        fillColor: palette[colorIndex],
        fillOpacity: 0.9,
      });
    });
    
    // Create roof segment outlines and labels
    // Group panels by segment index
    const segmentPanels = new Map<number, SolarPanel[]>();
    solarPotential.solarPanels.forEach(panel => {
      if (!segmentPanels.has(panel.segmentIndex)) {
        segmentPanels.set(panel.segmentIndex, []);
      }
      segmentPanels.get(panel.segmentIndex)?.push(panel);
    });
    
    // Get all unique segment indices
    const segmentIndices = Array.from(segmentPanels.keys());
    
    // Create a polygon for each roof segment
    segmentPanels.forEach((panels, segmentIndex) => {
      // Get the segment info
      const segmentInfo = solarPotential.roofSegmentStats[segmentIndex];
      
      // Instead of using the bounding box directly, we'll create a rotated polygon
      // based on the segment's azimuth
      const azimuth = segmentInfo.azimuthDegrees;
      
      // Estimate the size of the segment based on the number of panels
      // and their arrangement
      const panelPositions = panels.map(panel => ({
        lat: panel.center.latitude,
        lng: panel.center.longitude
      }));
      
      console.log(`Segment ${segmentIndex}: ${panels.length} panels, azimuth: ${azimuth}°`);
      
      // If we have panels in this segment, create a better outline
      if (panelPositions.length > 0) {
        // Find the center of the segment
        const center = {
          lat: segmentInfo.center.latitude,
          lng: segmentInfo.center.longitude
        };
        
        // Instead of using a simple square, we'll calculate a more accurate bounding shape
        // First, convert all panel positions to x,y coordinates relative to the center
        // This will allow us to find the actual dimensions of the segment
        const relativePositions = panels.map(panel => {
          const bearing = geometryLibrary.spherical.computeHeading(
            center,
            { lat: panel.center.latitude, lng: panel.center.longitude }
          );
          const distance = geometryLibrary.spherical.computeDistanceBetween(
            center,
            { lat: panel.center.latitude, lng: panel.center.longitude }
          );
          
          // Adjust the bearing by subtracting the azimuth to align with the roof orientation
          const adjustedBearing = bearing - azimuth;
          const radians = adjustedBearing * Math.PI / 180;
          
          // Calculate x,y coordinates (rotated to align with roof orientation)
          return {
            x: distance * Math.sin(radians),
            y: distance * Math.cos(radians)
          };
        });
        
        // Find the min/max x and y values to create a tight bounding box
        let minX = Infinity, maxX = -Infinity, minY = Infinity, maxY = -Infinity;
        
        relativePositions.forEach(pos => {
          minX = Math.min(minX, pos.x);
          maxX = Math.max(maxX, pos.x);
          minY = Math.min(minY, pos.y);
          maxY = Math.max(maxY, pos.y);
        });
        
        // Log size information for debugging
        const width = maxX - minX;
        const height = maxY - minY;
        console.log(`Segment ${segmentIndex} dimensions: ${width.toFixed(2)}m × ${height.toFixed(2)}m`);
        
        // Add a small padding (10%)
        const paddingX = (maxX - minX) * 0.1;
        const paddingY = (maxY - minY) * 0.1;
        
        minX -= paddingX;
        maxX += paddingX;
        minY -= paddingY;
        maxY += paddingY;
        
        // Ensure a minimum size for very small segments (like segment 7)
        const minSize = 2.0; // Minimum size in meters
        if (maxX - minX < minSize) {
          const center = (maxX + minX) / 2;
          minX = center - minSize/2;
          maxX = center + minSize/2;
        }
        if (maxY - minY < minSize) {
          const center = (maxY + minY) / 2;
          minY = center - minSize/2;
          maxY = center + minSize/2;
        }
        
        // Create the corners of the bounding box
        const corners = [
          { x: minX, y: minY }, // bottom left
          { x: maxX, y: minY }, // bottom right
          { x: maxX, y: maxY }, // top right
          { x: minX, y: maxY }, // top left
        ];
        
        // Create the polygon with rotated points
        const segmentPolygon = new google.maps.Polygon({
          paths: corners.map(({ x, y }) => {
            // Calculate the distance from center
            const distance = Math.sqrt(x * x + y * y);
            // Calculate the bearing in the rotated coordinate system
            let bearing = Math.atan2(x, y) * (180 / Math.PI);
            // Add back the azimuth to get the actual bearing
            bearing += azimuth;
            
            // Convert back to lat/lng
            return geometryLibrary.spherical.computeOffset(
              center,
              distance,
              bearing
            );
          }),
          strokeColor: '#FF0000',
          strokeOpacity: 1.0,
          strokeWeight: 4, // Increase stroke weight for better visibility
          fillColor: '#FF0000',
          fillOpacity: 0.15, // Slightly increase fill opacity
          zIndex: 1 // Make sure segments are below panels
        });
        
        // Store the segment index with the polygon for reference
        // @ts-ignore - Adding custom property
        segmentPolygon.segmentIndex = segmentIndex;
        
        // Create a label for the segment
        const label = new google.maps.Marker({
          position: center,
          label: {
            text: `${segmentIndex}`,
            color: 'white',
            fontSize: '16px',
            fontWeight: 'bold'
          },
          icon: {
            path: google.maps.SymbolPath.CIRCLE,
            scale: 12,
            fillColor: '#FF0000',
            fillOpacity: 1,
            strokeWeight: 0
          },
          zIndex: 2 // Make sure labels are above segments
        });
        
        // Store the segment index with the marker for reference
        // @ts-ignore - Adding custom property
        label.segmentIndex = segmentIndex;
        
        roofSegmentPolygons.push(segmentPolygon);
        roofSegmentLabels.push(label);
      }
    });
    
    // Use our helper functions to update visibility based on current toggle states
    setTimeout(() => {
      updatePanelVisibility();
      updateSegmentVisibility();
    }, 50);
  }

  $: showSolarPotential(location);
</script>

{#if requestError}
  <div class="error-container on-error-container-text">
    <Expandable section={title} icon="error" {title} subtitle={requestError.error.status}>
      <div class="grid place-items-center py-2 space-y-4">
        <div class="grid place-items-center">
          <p class="body-medium">
            Error on <code>buildingInsights</code> request
          </p>
          <p class="title-large">ERROR {requestError.error.code}</p>
          <p class="body-medium"><code>{requestError.error.status}</code></p>
          <p class="label-medium">{requestError.error.message}</p>
        </div>
        <md-filled-button role={undefined} on:click={() => showSolarPotential(location)}>
          Retry
          <md-icon slot="icon">refresh</md-icon>
        </md-filled-button>
      </div>
    </Expandable>
  </div>
{:else if !buildingInsights}
  <div class="grid py-8 place-items-center">
    <md-circular-progress four-color indeterminate />
  </div>
{:else if configId !== undefined && panelConfig}
  <Expandable
    bind:section={expandedSection}
    {icon}
    {title}
    subtitle={`Yearly energy: ${(
      (panelConfig.yearlyEnergyDcKwh * panelCapacityRatio) /
      1000
    ).toFixed(2)} MWh`}
  >
    <div class="flex flex-col space-y-2 px-2">
      <span class="outline-text label-medium">
        <b>{title}</b> provides data on the location, dimensions & solar potential of a building.
      </span>

      <InputPanelsCount
        bind:configId
        solarPanelConfigs={buildingInsights.solarPotential.solarPanelConfigs}
      />
      <NumberInput
        bind:value={panelCapacityWatts}
        icon="bolt"
        label="Panel capacity"
        suffix="Watts"
      />
      
      <!-- Integrated visualization controls -->
      <div class="border-t border-outline my-2"></div>
      <h3 class="title-small">Visualization Options</h3>
      
      <InputBool 
        bind:value={showPanels} 
        label="Show Solar Panels" 
        onChange={() => {
          // Make sure the toggle state is properly applied
          setTimeout(() => {
            updatePanelVisibility();
          }, 0);
        }}
      />
      <InputBool 
        bind:value={disableAnimation} 
        label="Show All Panels (No Animation)" 
        disabled={!showPanels}
        onChange={() => {
          // Make sure the toggle state is properly applied
          setTimeout(() => {
            updatePanelVisibility();
          }, 0);
        }}
      />
      <div class="border-t border-outline my-2"></div>
      <h3 class="title-small">Roof Segments</h3>
      <InputBool 
        bind:value={showSegments} 
        label="Show Roof Segments"
        onChange={() => {
          // Make sure the toggle state is properly applied
          setTimeout(() => {
            updateSegmentVisibility();
          }, 0);
        }}
      />
      
      <div class="flex flex-col">
        <label class="label-medium mb-1">Filter Segments (e.g. "0,1,2")</label>
        <input 
          type="text" 
          bind:value={segmentFilter} 
          class="px-3 py-2 border border-outline rounded-md"
          placeholder="Enter segment numbers"
          disabled={!showSegments}
        />
        <span class="label-small mt-1 text-outline">Comma-separated segment numbers</span>
      </div>
      
      <div class="grid justify-items-end mt-2">
        <md-filled-tonal-button role={undefined} on:click={() => apiResponseDialog.show()}>
          API response
        </md-filled-tonal-button>
      </div>

      <md-dialog bind:this={apiResponseDialog}>
        <div slot="headline">
          <div class="flex items-center primary-text">
            <md-icon>{icon}</md-icon>
            <b>&nbsp;{title}</b>
          </div>
        </div>
        <div slot="content">
          <Show value={buildingInsights} label="buildingInsightsResponse" />
        </div>
        <div slot="actions">
          <md-text-button role={undefined} on:click={() => apiResponseDialog.close()}>
            Close
          </md-text-button>
        </div>
      </md-dialog>
    </div>
  </Expandable>

  {#if expandedSection == title}
    <div class="absolute top-0 left-0 w-72">
      <div class="flex flex-col space-y-2 m-2">
        <SummaryCard
          {icon}
          {title}
          rows={[
            {
              icon: 'wb_sunny',
              name: 'Annual sunshine',
              value: showNumber(buildingInsights.solarPotential.maxSunshineHoursPerYear),
              units: 'hr',
            },
            {
              icon: 'square_foot',
              name: 'Roof area',
              value: showNumber(buildingInsights.solarPotential.wholeRoofStats.areaMeters2),
              units: 'm²',
            },
            {
              icon: 'solar_power',
              name: 'Max panel count',
              value: showNumber(buildingInsights.solarPotential.solarPanels.length),
              units: 'panels',
            },
            {
              icon: 'co2',
              name: 'CO₂ savings',
              value: showNumber(buildingInsights.solarPotential.carbonOffsetFactorKgPerMwh),
              units: 'Kg/MWh',
            },
          ]}
        />

        <div class="p-4 w-full surface on-surface-text rounded-lg shadow-md">
          <div class="flex justify-around">
            <Gauge
              icon="solar_power"
              title="Panels count"
              label={showNumber(panelConfig.panelsCount)}
              labelSuffix={`/ ${showNumber(solarPanels.length)}`}
              max={solarPanels.length}
              value={panelConfig.panelsCount}
            />

            <Gauge
              icon="energy_savings_leaf"
              title="Yearly energy"
              label={showNumber((panelConfig?.yearlyEnergyDcKwh ?? 0) * panelCapacityRatio)}
              labelSuffix="KWh"
              max={buildingInsights.solarPotential.solarPanelConfigs.slice(-1)[0]
                .yearlyEnergyDcKwh * panelCapacityRatio}
              value={panelConfig.yearlyEnergyDcKwh * panelCapacityRatio}
            />
          </div>
        </div>
      </div>
    </div>
  {/if}
{/if}
