<template>
  <v-col cols="12" md="8" lg="5">
    <v-card v-if="!isLoaded" title="Loading Project Screens..." :color="color2">
    <v-progress-linear color="#1428a0" indeterminate rounded />
    </v-card>
    <v-card v-else :class="{ 'dark-theme2': darkMode }">
      <v-card-title class="d-flex align-center">
        <v-icon class="mr-2">mdi-monitor-dashboard</v-icon>
        Screens
      </v-card-title>
      <v-card-text>
        <!-- Info Panel -->
        <v-alert id="info-screen" density="compact" variant="tonal" type="info">
          <span class="special-color">Scheme used by this project:</span> {{ screensData.issueTypeScreenScheme }}
          <v-chip class="aui-lozenge ml-2" v-if="screensData.default_screen === screensData.issueTypeScreenScheme">
            Default
          </v-chip>
        </v-alert>
        <br>
        <!-- Expandable Panels -->
        <v-expansion-panels :class="{ 'dark-theme2': darkMode }" id="screen-schemes" multiple v-model="openPanels">
          <v-expansion-panel v-for="(scheme, index) in screensData.issueTypeScreenDetails" :key="index" :value="index">
            <v-expansion-panel-title>
              <strong>{{ scheme.screenSchemeName }}</strong>
              <v-chip class="aui-lozenge default-scheme ml-2" v-if="screensData.default_screen === scheme.screenSchemeName">
                  Default
                  <v-tooltip text="Default screen scheme" origin="auto" :content-class="{ 'custom-tooltip': darkMode}" :contained = "true" location="top" activator="parent"></v-tooltip>
              </v-chip>
              <v-chip class="aui-lozenge used-by ml-2" v-if="scheme.shared" @click.prevent.stop="openDialog($event, index)">
                <em>
                  Used by 
                  <a href="#" @click.prevent.stop="openDialog($event, index)">
                    {{ scheme.projects.split(', ').length }} projects
                  </a>
                </em>
              </v-chip>
            </v-expansion-panel-title>
            <v-divider></v-divider>
            <!-- New Horizontal Panel -->
            <div v-show="!openPanels.includes(index)" class="horizontal-issue-types">
              <div v-for="(issueType, i) in scheme.issueType.split(', ')" :key="i" class="d-flex align-center" style="font-size: 14px !important;">
                <v-img :src="`http://s2jira2:8081/${scheme.issueTypeUrls.split(', ')[i]}`" width="16" height="16" class="mr-2"></v-img>
                {{ issueType }}
                <v-chip class="aui-lozenge default-scheme ml-2" v-if="screensData.default_issuetype  === issueType">
                Default
                <v-tooltip text="Default issue type" origin="auto" :content-class="{ 'custom-tooltip': darkMode}" :contained = "true" location="top" activator="parent"></v-tooltip>
              </v-chip>
              </div>
            </div>
            <v-expansion-panel-text>
              <div class="d-flex justify-space-between">
                <div>
                  {{ scheme.issueType.split(', ').length > 1 ? `These ${scheme.issueType.split(', ').length} issue types...` : `This issue type...` }}
                  <v-list-item v-for="(issueType, i) in scheme.issueType.split(', ')" :key="i" style="display: flex !important">
                    <div class="d-flex align-center" style="font-size: 14px !important;">
                      <v-img :src="`http://s2jira2:8081/${scheme.issueTypeUrls.split(', ')[i]}`" width="16" height="16" class="mr-2"></v-img>
                      {{ issueType }}
                      <v-chip class="aui-lozenge default-scheme ml-2" v-if="screensData.default_issuetype  === issueType">
                      Default
                      <v-tooltip style="font-weight: bold" text="Default issue type" origin="top left" :content-class="{ 'custom-tooltip': darkMode}" location="end" activator="parent"></v-tooltip>
                      </v-chip>
                    </div>
                  </v-list-item>
                </div>
                <div>
                  {{ scheme.issueType.split(', ').length > 1 ? `…use this screen scheme` : `...uses this screen scheme` }}
                  <v-table density="compact" class="bordered-table">
                    <thead>
                      <tr>
                        <th class="text-left"><strong>Operation</strong></th>
                        <th class="text-left"><strong>Screen</strong></th>
                      </tr>
                    </thead>
                    <tbody>
                    <template v-for="(screen, k) in scheme.screens" :key="`screen-${k}`">
                    <tr v-for="(operation, j) in screen.operationName.split(', ')" :key="`${k}-${j}`">
                      <td>{{ operation }} Issue</td>
                      <td>{{ screen.screenName }}</td>
                    </tr>
                  </template>
                    </tbody>
                  </v-table>
                </div>
              </div>
            </v-expansion-panel-text>
          </v-expansion-panel>
        </v-expansion-panels>

        <!-- Dialog for Projects -->
        <aui-inline-dialog v-if="isDialogOpen" ref="projectDialog" class="shared-items-target shared-projects-target aui-layer aui-alignment-side-bottom aui-alignment-snap-center"
                           :alignment="dialogAlignment"
                           aria-label="View associated items" resolved="" tabindex="0" role="dialog" open=""
                           :style="{ zIndex: 3000, position: 'absolute', top: dialogPosition.top + 'px', left: dialogPosition.left + 'px', margin: '0px' }"
                           :data-popper-placement="dialogPlacement"
                           x-placement="top">
          <div class="aui-inline-dialog-contents">
            <div class="shared-items-content">
              <h4 class="mb-2">Projects using this screen:</h4>
              <ul class="shared-items-list shared-projects-list">
                <li class="shared-item-name shared-project-name" v-for="(project, i) in visibleProjects" :key="i">
                  {{ project }}
                </li>
              </ul>
              <span v-if="hiddenProjectCount > 0" class="hidden-projects">
                and {{ hiddenProjectCount }} hidden projects
              </span>
            </div>
          </div>
        </aui-inline-dialog>
      </v-card-text>
    </v-card>
  </v-col>
</template>

<script setup>
import { ref, nextTick, computed, watch, onMounted, onUnmounted } from "vue";
import { useScreenStore } from "@/stores/screens";
import { store } from "@/stores/dark";
import { useRouter } from 'vue-router';

const screenStore = useScreenStore();
const router = useRouter();
const screensData = ref({});
const darkMode = computed(() => store.state.darkMode);
const isDialogOpen = ref(false); // Boolean to control dialog visibility
const selectedPanelIndex = ref(null); // Index of the selected panel
const projectDialog = ref(null); // Ref to the dialog element
const dialogPosition = ref({ top: 0, left: 0 }); // Position of the dialog
const openPanels = ref([]);
const dialogPlacement = ref('bottom'); // Default placement
const dialogAlignment = ref('bottom center'); // Default alignment
const darkColor = "black";
const textColor = "white";
const color2 = computed(() => (darkMode.value ? darkColor : textColor));
const isLoaded = ref(false);

async function fetchScreenData() {
  try {
    const projectKey = router.currentRoute.value.params.projectKey;
    const response = await screenStore.fetchScreenData(projectKey);
    screensData.value = response;
    isLoaded.value = true;
    console.log("SCREEENS", screensData.value)
  } catch (error) {
    console.error("Error retrieving screens:", error);
  }
}

function openDialog(event, index) {
  selectedPanelIndex.value = index;
  isDialogOpen.value = true;

  nextTick(() => {
    const anchor = event.currentTarget || event.target;
    const rect = anchor.getBoundingClientRect();

    const dialogEl = projectDialog.value;
    const parent = dialogEl.offsetParent || document.body;
    const parentRect = parent.getBoundingClientRect();

    const dialogWidth = dialogEl.offsetWidth;
    const dialogHeight = dialogEl.offsetHeight;

    // Convert viewport coords -> parent coords
    let top = (rect.bottom - parentRect.top) + parent.scrollTop + 10;
    let left = (rect.left - parentRect.left) + parent.scrollLeft + (rect.width / 2) - (dialogWidth / 2);

    // If it overflows parent vertically, flip above
    const parentHeight = parent.clientHeight;
    if (top + dialogHeight > parentHeight - 8) {
      top = (rect.top - parentRect.top) + parent.scrollTop - dialogHeight - 10;
      dialogPlacement.value = 'top'; // Update placement
      dialogAlignment.value = 'top center'; // Update alignment
    } else {
      dialogPlacement.value = 'bottom'; // Reset placement
      dialogAlignment.value = 'bottom center'; // Reset alignment
    }

    // Clamp horizontally within parent
    left = Math.max(8, Math.min(left, parent.clientWidth - dialogWidth - 8));

    dialogPosition.value = { top, left };
  });
}

function closeDialog() {
  isDialogOpen.value = false;
}

function handleClickOutside(event) {
  if (isDialogOpen.value && projectDialog.value && !projectDialog.value.contains(event.target)) {
    closeDialog();
  }
}

const visibleProjects = computed(() => {
  if (selectedPanelIndex.value !== null && screensData.value.issueTypeScreenDetails) {
    const projectsArray = screensData.value.issueTypeScreenDetails[selectedPanelIndex.value].projects.split(', ');
    return projectsArray.slice(0, 15);
  }
  return [];
});

const hiddenProjectCount = computed(() => {
  if (selectedPanelIndex.value !== null && screensData.value.issueTypeScreenDetails) {
    const projectsArray = screensData.value.issueTypeScreenDetails[selectedPanelIndex.value].projects.split(', ');
    return Math.max(0, projectsArray.length - 15);
  }
  return 0;
});

// Watch for route changes to fetch new data
watch(
  () => router.currentRoute.value.params.projectKey,
  (newProjectKey) => {
    if (newProjectKey) {
      fetchScreenData();
    }
  },
  { immediate: true }
);

// Attach event listener on component mount
onMounted(() => {
  document.addEventListener('click', handleClickOutside);
});

// Remove event listener on component unmount
onUnmounted(() => {
  document.removeEventListener('click', handleClickOutside);
});
</script>


<style>

#info-screen i {
  font-size: 16px !important;
  width: 0px !important;
  color: #0052cc !important;
}

#info-screen .v-alert__content {
  color: black !important;
}

#info-screen.v-alert {
  max-width: max-content;
  background-color: #deebff;
}

.v-chip.aui-lozenge {
  background: #42526e;
  color: white;
  border: 0;
  border-radius: 3px !important;
  display: inline-flex;
  font-size: 11px;
  font-weight: 700;
  line-height: 1;
  margin: 0;
  padding: 2px 5px;
  text-align: center;
  text-decoration: none;
  text-transform: uppercase;
  --v-chip-height: 20px;
  overflow: visible;
}

.aui-lozenge.default-scheme:hover {
  cursor: pointer;
}

.aui-lozenge.used-by {
  background-color: #deebff;
  color: #42526e;
}

.aui-lozenge.used-by > em {
  font-style: normal;
}

.aui-lozenge.used-by > em > a {
  color: #0052cc;
  text-decoration: none;
}

.dark-theme2 .aui-lozenge.used-by > em > a {
  color: white;
  text-decoration: underline;
}

.aui-lozenge.used-by > em > a:hover {
  text-decoration: underline;
}

.aui-lozenge .v-chip__underlay {
  background: #42526e !important;
  top: unset;
}

.dark-theme2 .aui-lozenge.used-by {
  color: white;
}

#screen-schemes.v-expansion-panels {
  gap: 5px;
}

#screen-schemes.dark-theme2 .v-expansion-panel {
    background-color: #424242;
    color: white;
}

#screen-schemes.v-expansion-panels:not(.v-expansion-panels--variant-accordion) > 
:first-child:not(:last-child):not(.v-expansion-panel--active):not(.v-expansion-panel--before-active) {
  border-bottom-left-radius: 4px !important;
  border-bottom-right-radius: 4px !important;
}

#screen-schemes.v-expansion-panels:not(.v-expansion-panels--variant-accordion) > 
:last-child:not(:first-child):not(.v-expansion-panel--active):not(.v-expansion-panel--after-active) {
  border-top-left-radius: 4px !important;
  border-top-right-radius: 4px !important;
}

.bordered-table {
  border: 1px solid #ccc; /* Adjust the border color as needed */
  border-radius: 4px; /* Optional: Add rounded corners */
}

.dark-theme2 .bordered-table {
  border: 1px solid white; /* Adjust the border color as needed */
  border-radius: 4px; /* Optional: Add rounded corners */
}

th.text-left {
  border-bottom: 2px solid #ccc !important; /* Thicker border for header cells */
}

.dark-theme2 th.text-left {
  border-bottom: 2px solid white !important; /* Thicker border for header cells */
  color: white !important
}

.dark-theme2 .v-table.bordered-table {
  background:  #424242 !important;
  color: white !important;
}

.dark-theme2 .v-table .v-table__wrapper > table > tbody > tr:not(:last-child) > td {
  border-bottom: thin solid rgb(255 255 255 / 12%) !important;
}

.horizontal-issue-types {
  display: flex;
  flex-wrap: wrap;
  background-color: #fcfcfc; /* Light background to match the expansion panel */
  padding: 16px 24px; /* Padding to match the expansion panel */
  box-sizing: border-box;
}

.dark-theme2 .horizontal-issue-types {
  background-color: #424242 !important;
}

.horizontal-issue-types .d-flex.align-center {
  margin-right: 16px; /* Space between issue types */
}

.special-color {
  color: #344563;
}

/* Custom styles to match the provided HTML */
aui-inline-dialog {
  --aui-inline-dialog-border-width: 0px;
  --aui-tail-x: 0;
  --aui-tail-w: 8px;
  --aui-tail-bw: calc(0px + 1px);
  display: block;
}

aui-inline-dialog.aui-layer[open] {
  opacity: 1;
  transition: opacity .2s;
  transition-delay: 0s;
  visibility: visible;
}

aui-inline-dialog.aui-alignment-side-bottom[data-popper-placement*=top], aui-inline-dialog.aui-alignment-side-top {
  --aui-tail-deg: 180deg;
  --aui-tail-y: 8px;
}

aui-inline-dialog.aui-alignment-side-bottom {
  --aui-tail-deg: 0deg;
  --aui-tail-y: 8px;
}

aui-inline-dialog:before {
  border-bottom-color: #dfe1e6;
}

aui-inline-dialog:after, aui-inline-dialog:before {
  border: var(--aui-tail-w) solid transparent;
  border-bottom: var(--aui-tail-w) solid #fff;
  box-sizing: border-box;
  content: "";
  display: inline-block;
  height: 0;
  pointer-events: none;
  position: absolute;
  width: 0;
  transform: rotate(var(--aui-tail-deg));
}

aui-inline-dialog.aui-alignment-side-bottom[data-popper-placement*=top]:after, aui-inline-dialog.aui-alignment-side-top:after {
  top: calc(100% - var(--aui-tail-w) - var(--aui-tail-bw));
}

aui-inline-dialog.aui-alignment-side-bottom:after, aui-inline-dialog.aui-alignment-side-top[data-popper-placement*=bottom]:after {
  top: calc(0% - var(--aui-tail-w) + var(--aui-tail-bw));
}

aui-inline-dialog .aui-inline-dialog-contents {
  box-shadow: var(--box-shadow);
  background: #fff;
  border: 0px solid #dfe1e6;
  border-radius: 3px;
  padding: 20px;
  margin: 8px 0px;
  overflow: hidden;
}

aui-inline-dialog[data-popper-placement="bottom"] .aui-inline-dialog-contents {
  --box-shadow: 0 -4px 8px -2px rgba(9, 30, 66, 0.25), 0 0 1px rgba(9, 30, 66, 0.25);
}

aui-inline-dialog[data-popper-placement="top"] .aui-inline-dialog-contents {
  --box-shadow: 0 4px 8px -2px rgba(9, 30, 66, 0.25), 0 0 1px rgba(9, 30, 66, 0.25);
}

aui-inline-dialog.aui-alignment-side-bottom[data-popper-placement*=top]::before{
  display: none;

}

aui-inline-dialog.aui-alignment-snap-center:after, aui-inline-dialog.aui-alignment-snap-center:before {
left: calc(50% - var(--aui-tail-w));
}

.shared-items-target .aui-inline-dialog-contents, .shared-item-target .aui-inline-dialog-contents {
  padding: 0;
}

.shared-items-content, .shared-item-content {
  padding: 16px;
}

.dark-theme2 .shared-items-content {
  color: black;
}

.shared-items-list {
  display: block;
  margin: 0;
  list-style: none;
  padding: 0;
  max-height: 10em;
  overflow-y: auto;
}

aui-inline-dialog ::-webkit-scrollbar-thumb {
    background: #8b8b8b;
    border-radius: 3px;
}

aui-inline-dialog ::-webkit-scrollbar-thumb:hover {
    background: #636363;
    border-radius: 3px;
}

aui-inline-dialog ::-webkit-scrollbar-thumb:active {
    background: #636363;
    border-radius: 3px;
}

aui-inline-dialog ::-webkit-scrollbar-thumb:focus {
    background: #636363;
    border-radius: 3px;
}

.shared-items-list li {
  text-align: start;
}


.v-overlay__content.custom-tooltip {
  color: #424242 !important;
  background-color: rgba(234, 234, 234, 0.90) !important;
}
</style>

<style scoped>
.v-card {
  transition: transform 0.2s;
}

.v-card:hover {
  transform: translateY(-2px);
}

</style>
