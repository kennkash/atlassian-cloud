<template>
    <section :class="darkMode ? 'dark-mode-fields intro-section' : 'fields intro-section'">
        <div class="flex-wrapper">
            <v-icon :style="{ color: color, marginRight: '10px' }">
                mdi-bug
            </v-icon>
            <h2 :class="['intro-title', { 'dark-text': darkMode }]">Issue Types</h2>
        </div>
        <p :class="['intro-text', { 'dark-text': darkMode }]">
            Jira organizes issue types into <strong>two primary</strong> families, <em>Standard</em> and <em>Sub-Task</em> :
        </p>
        <ol :class="['intro-list', { 'dark-text': darkMode }]">
            <li>
                <strong>Standard:</strong>
                These are the “top‑level” work items that can exist on their own.
                They represent the main units of value you track in your projects.
            </li>
            <li>
                <strong>Sub‑Task:</strong>
                Child issues that break a parent Standard issue into smaller, more manageable pieces.
                Sub‑Tasks inherit the parent’s context and are limited to the lifecycle of that parent.
            </li>
        </ol>
    </section>
    <br>
    <v-card v-if="!isLoaded" title="Loading Issue Types..." :color="color2">
        <v-progress-linear color="#1428a0" indeterminate rounded />
    </v-card>
    <v-card v-else :loading="false" :class="{ 'dark-theme2': darkMode }">
        <v-data-table :headers="issueTypeHeaders" :items="issueTypeData" dense :items-per-page="10">
            <!-- Name column (icon & text)-->
            <template v-slot:item.name="{ item }">
                <div class="d-flex align-center">
                    <v-img height="16" width="16" :src="item.raw.iconUrl" :alt="item.raw.name" class="mr-2"
                        style="display: flex; flex:none"></v-img>
                    {{ item.raw.name }}
                </div>
            </template>
            <!-- Description column with fallback -->
            <template v-slot:item.description="{ item }">
                <span>
                    {{ (item.raw.description && item.raw.description.trim())
                        ? item.raw.description
                        : 'No description' }}
                </span>
            </template>
        </v-data-table>
    </v-card>
</template>


<script setup>
import { ref, computed, watch, onMounted } from "vue";
import { useIssueTypeStore } from "@/stores/issuetype";
import { store } from "@/stores/dark";

const IssueTypeStore = useIssueTypeStore();
const issueTypeData = ref([]);
const darkMode = computed(() => store.state.darkMode);
const darkColor = "black";
const textColor = "white";
const color = computed(() => (darkMode.value ? "white" : "black"));
const color2 = computed(() => (darkMode.value ? darkColor : textColor));
const isLoaded = ref(false);

async function fetchITData() {
    try {
        const response = await IssueTypeStore.fetchAllIssueTypes();
        issueTypeData.value = response;
        isLoaded.value = true;
        console.log("issueTypeData: ", issueTypeData.value);
    } catch (error) {
        console.error("Error retrieving issue types:", error);
    }
}


const issueTypeHeaders = ref([
    { title: 'ID', key: 'id' },
    { title: 'Name', key: 'name' },
    { title: 'Description', key: 'description' },
    { title: 'Type', key: 'type' },
])

onMounted(async () => {
    await fetchITData();
});
</script>

<style scoped>
.intro-section {
    padding: 20px;
    border-radius: 8px;
    background-color: #f5f5f5;
    margin-bottom: 20px;
}

.intro-title {
    font-size: 24px;
    font-weight: 600;
    margin: 0;
}

.intro-text {
    font-size: 16px;
    line-height: 1.6;
    margin: 10px 0;
}

.intro-list {
    list-style-position: inside;
    margin: 0 auto; 
    display: inline-block;
    padding-left: 20%;
}

.intro-list li {
    margin: 5px 0;
    font-size: 15px;
    line-height: 1.5;
    text-indent: -10px;
    text-align: start;
    /* Adjust this value as needed */
}

.dark-text {
    color: #cfd8dc;
}

.dark-mode-fields {
    background-color: #333;
    color: #cfd8dc;
}
</style>
