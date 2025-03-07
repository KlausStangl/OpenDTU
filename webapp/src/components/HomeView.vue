<template>
    <div class="container-fluid" role="main">
        <div class="page-header">
            <h1>Live Data</h1>
        </div>

        <div class="text-center" v-if="dataLoading">
            <div class="spinner-border" role="status">
                <span class="visually-hidden">Loading...</span>
            </div>
        </div>

        <template v-else>
            <div class="row gy-3">
                <div class="col-sm-3 col-md-2">
                    <div class="nav nav-pills row-cols-sm-1" id="v-pills-tab" role="tablist"
                        aria-orientation="vertical">
                        <button v-for="inverter in inverterData" :key="inverter.serial" class="nav-link"
                            :id="'v-pills-' + inverter.serial + '-tab'" data-bs-toggle="pill"
                            :data-bs-target="'#v-pills-' + inverter.serial" type="button" role="tab"
                            aria-controls="'v-pills-' + inverter.serial" aria-selected="true">
                            {{ inverter.name }}
                        </button>
                    </div>
                </div>

                <div class="tab-content col-sm-9 col-md-10" id="v-pills-tabContent">
                    <div v-for="inverter in inverterData" :key="inverter.serial" class="tab-pane fade show"
                        :id="'v-pills-' + inverter.serial" role="tabpanel"
                        :aria-labelledby="'v-pills-' + inverter.serial + '-tab'" tabindex="0">
                        <div class="card">
                            <div class="card-header text-white bg-primary d-flex justify-content-between align-items-center"
                                :class="{
                                    'bg-danger': inverter.age_critical,
                                    'bg-primary': !inverter.age_critical,
                                }">
                                {{ inverter.name }} (Inverter Serial Number:
                                {{ inverter.serial }}) (Data Age:
                                {{ inverter.data_age }} seconds)

                                <button v-if="inverter.events >= 0" type="button"
                                    class="btn btn-sm btn-secondary position-relative"
                                    @click="onShowEventlog(inverter.serial)"
                                    title="Show Eventlog">
                                    <BIconJournalText style="font-size:24px;" />
                                    <span
                                        class="position-absolute top-0 start-100 translate-middle badge rounded-pill bg-danger">
                                        {{ inverter.events }}
                                        <span class="visually-hidden">unread messages</span>
                                    </span>
                                </button>
                            </div>
                            <div class="card-body">
                                <div class="row flex-row-reverse flex-wrap-reverse align-items-end g-3">
                                    <template v-for="channel in 5" :key="channel">
                                        <div v-if="inverter[channel - 1]" :class="`col order-${5 - channel}`">
                                            <InverterChannelInfo
                                                :channelData="inverter[channel - 1]"
                                                :channelNumber="channel - 1" />
                                        </div>
                                    </template>
                                </div>
                            </div>
                        </div>
                    </div>
                </div>
            </div>
        </template>

        <div class="modal" id="eventView" tabindex="-1">
            <div class="modal-dialog modal-lg">
                <div class="modal-content">
                    <div class="modal-header">
                        <h5 class="modal-title">Event Log</h5>
                        <button type="button" class="btn-close" data-bs-dismiss="modal" aria-label="Close"></button>
                    </div>
                    <div class="modal-body">
                        <div class="text-center" v-if="eventLogLoading">
                            <div class="spinner-border" role="status">
                                <span class="visually-hidden">Loading...</span>
                            </div>
                        </div>

                        <EventLog v-if="!eventLogLoading" :eventLogList="eventLogList" />
                    </div>

                    <div class="modal-footer">
                        <button type="button" class="btn btn-secondary" @click="onHideEventlog"
                            data-bs-dismiss="modal">Close</button>
                    </div>

                </div>
            </div>
        </div>

    </div>
</template>

<script lang="ts">
import { defineComponent } from 'vue';
import InverterChannelInfo from "@/components/partials/InverterChannelInfo.vue";
import * as bootstrap from 'bootstrap';
import EventLog from '@/components/partials/EventLog.vue';

declare interface Inverter {
    serial: number,
    name: string,
    age_critical: boolean,
    data_age: 0,
    events: 0
}

export default defineComponent({
    components: {
        InverterChannelInfo,
        EventLog
    },
    data() {
        return {
            socket: {} as WebSocket,
            heartInterval: 0,
            dataAgeInterval: 0,
            dataLoading: true,
            inverterData: [] as Inverter[],
            isFirstFetchAfterConnect: true,
            eventLogView: {} as bootstrap.Modal,
            eventLogList: {},
            eventLogLoading: true
        };
    },
    created() {
        this.getInitialData();
        this.initSocket();
        this.initDataAgeing();
    },
    mounted() {
        this.eventLogView = new bootstrap.Modal('#eventView');
    },
    unmounted() {
        this.closeSocket();
    },
    updated() {
        // Select first tab
        if (this.isFirstFetchAfterConnect) {
            this.isFirstFetchAfterConnect = false;

            const firstTabEl = this.$el.querySelector(
                "#v-pills-tab:first-child button"
            );
            if (firstTabEl != null) {
                const firstTab = new bootstrap.Tab(firstTabEl);
                firstTab.show();
            }
        }
    },
    methods: {
        getInitialData() {
            this.dataLoading = true;
            fetch("/api/livedata/status")
                .then((response) => response.json())
                .then((data) => {
                    this.inverterData = data;
                    this.dataLoading = false;
                });
        },
        initSocket() {
            console.log("Starting connection to WebSocket Server");

            const { protocol, host } = location;
            const webSocketUrl = `${protocol === "https:" ? "wss" : "ws"
                }://${host}/livedata`;

            this.socket = new WebSocket(webSocketUrl);

            this.socket.onmessage = (event) => {
                console.log(event);
                this.inverterData = JSON.parse(event.data);
                this.dataLoading = false;
                this.heartCheck(); // Reset heartbeat detection
            };

            this.socket.onopen = function (event) {
                console.log(event);
                console.log("Successfully connected to the echo websocket server...");
            };

            // Listen to window events , When the window closes , Take the initiative to disconnect websocket Connect
            window.onbeforeunload = () => {
                this.closeSocket();
            };
        },
        initDataAgeing() {
            this.dataAgeInterval = setInterval(() => {
                this.inverterData.forEach(element => {
                    element.data_age++;
                });
            }, 1000);
        },
        // Send heartbeat packets regularly * 59s Send a heartbeat
        heartCheck() {
            this.heartInterval && clearTimeout(this.heartInterval);
            this.heartInterval = setInterval(() => {
                if (this.socket.readyState === 1) {
                    // Connection status
                    this.socket.send("ping");
                } else {
                    this.initSocket(); // Breakpoint reconnection 5 Time
                }
            }, 59 * 1000);
        },
        /** To break off websocket Connect */
        closeSocket() {
            this.socket.close();
            this.heartInterval && clearTimeout(this.heartInterval);
            this.isFirstFetchAfterConnect = true;
        },
        onHideEventlog() {
            this.eventLogView.hide();
        },
        onShowEventlog(serial: number) {
            this.eventLogLoading = true;
            fetch("/api/eventlog/status")
                .then((response) => response.json())
                .then((data) => {
                    this.eventLogList = data[serial];
                    this.eventLogLoading = false;
                });

            this.eventLogView.show();
        },
    },
});
</script>