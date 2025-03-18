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
  import type { MdSwitch } from '@material/web/switch/switch';

  export let label: string;
  export let value: boolean = false;
  export let onChange: (value: boolean) => void = () => {};
  export let disabled: boolean = false;

  function onClick(event: Event) {
    if (disabled) return;
    
    // Instead of preventing default, let's wait for the material component to update
    setTimeout(() => {
      // Get the current target state
      const target = event.target as MdSwitch;
      
      // Update our value to match the switch state
      value = target.selected;
      
      // Call the onChange handler with the new value
      onChange(value);
    }, 0);
  }
</script>

<label for={label} class="p-2 relative inline-flex items-center cursor-pointer">
  <md-switch id={label} role={undefined} selected={value} on:click={onClick} disabled={disabled} />
  <span class="ml-3 body-large">{label}</span>
</label>
