<style>
    .heater-row .vertical_align_center {
        margin: auto 0;
    }

    .colHeaterIcons {
        width: 68px;
    }

    .canvasjs-chart-tooltip>div {
        border-radius: 10px !important;
    }

</style>

<template>
    <div>
        <v-card>
            <v-toolbar flat dense>
                <v-toolbar-title>
                    <span class="subheading"><v-icon left>mdi-thermometer-lines</v-icon>Temperatures</span>
                </v-toolbar-title>
                <v-spacer></v-spacer>
                <v-item-group class="v-btn-toggle" name="controllers">
                    <v-menu :offset-y="true" title="Preheat" v-if="this['gui/getPreheatPresets'].length">
                        <template v-slot:activator="{ on, attrs }">
                            <v-btn small class="px-2 minwidth-0" color="primary" v-bind="attrs" v-on="on" :disabled="['printing', 'paused'].includes(printer_state)">Presets <v-icon small>mdi-menu-down</v-icon></v-btn>
                        </template>
                        <v-list dense class="py-0">
                            <v-list-item v-for="preset of this['gui/getPreheatPresets']" v-bind:key="preset.index" link @click="preheat(preset)">
                                <v-list-item-icon class="mr-0">
                                    <v-icon small>mdi-fire</v-icon>
                                </v-list-item-icon>
                                <v-list-item-content>
                                    <v-list-item-title v-text="preset.name"></v-list-item-title>
                                </v-list-item-content>
                            </v-list-item>
                        </v-list>
                        <v-divider></v-divider>
                        <v-list dense class="py-0">
                            <v-list-item link @click="cooldown()">
                                <v-list-item-icon class="mr-0">
                                    <v-icon small>mdi-snowflake</v-icon>
                                </v-list-item-icon>
                                <v-list-item-content>
                                    <v-list-item-title>Cooldown</v-list-item-title>
                                </v-list-item-content>
                            </v-list-item>
                        </v-list>
                    </v-menu>
                    <v-btn small class="px-2 minwidth-0" color="primary" @click="cooldown()" v-if="this['gui/getPreheatPresets'].length === 0"><v-icon small class="mr-1">mdi-snowflake</v-icon>Cooldown</v-btn>
                    <v-menu :offset-y="true" :close-on-content-click="false" title="Setup Temperatures">
                        <template v-slot:activator="{ on, attrs }">
                            <v-btn small class="px-2 minwidth-0" color="grey darken-3" v-bind="attrs" v-on="on"><v-icon small>mdi-cog</v-icon></v-btn>
                        </template>
                        <v-list>
                            <v-list-item class="minHeight36">
                                <v-checkbox class="mt-0" v-model="boolTempchart" hide-details label="Show Chart"></v-checkbox>
                            </v-list-item>
                            <v-list-item class="minHeight36">
                                <v-checkbox class="mt-0" v-model="autoscaleTempchart" hide-details label="Autoscale Chart"></v-checkbox>
                            </v-list-item>
                        </v-list>
                    </v-menu>
                </v-item-group>
            </v-toolbar>
            <v-card-text class="px-0 py-2 content">
                <v-container class="px-0">
                    <v-row align="center">
                        <v-col class="py-2 font-weight-bold" style="padding-left: 68px;">Name</v-col>
                        <v-col class="py-2 text-center font-weight-bold d-none d-sm-block">State</v-col>
                        <v-col class="py-2 text-center font-weight-bold">Current</v-col>
                        <v-col class="py-2 pr-8 text-center font-weight-bold">Target</v-col>
                    </v-row>
                    <div v-for="(heater, index) in heaters" v-bind:key="index" >
                        <v-divider class="my-2"></v-divider>
                        <v-row align="center">
                            <v-col class="pl-8 pr-0 flex-grow-0 py-2 colHeaterIcons">
                                <v-icon :color="heater.color">mdi-{{ heater.icon }}</v-icon>
                            </v-col>
                            <v-col class="py-2 font-weight-bold"><span style="cursor: pointer;" @click="openHeater(heater)">{{ convertName(heater.name) }}</span></v-col>
                            <v-col class="py-2 text-center d-none d-sm-block"><small>{{ heater.target > 0 ? (heater.power !== null ? (heater.power > 0 ? (heater.power * 100).toFixed(0)+'%' : "0%") : "active") : "off" }}</small></v-col>
                            <v-col class="py-2 text-center">
                                <span class="d-block">{{ heater.temperature ? heater.temperature.toFixed(1) : 0 }}°C</span>
                                <span v-for="(values, key) of heater.additionValues" v-bind:key="key" class="d-block">{{ values.value.toFixed(1) }} {{ values.unit }}</span>
                            </v-col>
                            <v-col class="text-center py-2 pr-8 vertical_align_center">
                                <toolInput :name="heater.name" :target="heater.target" :min_temp="heater.min_temp" :max_temp="heater.max_temp" :items="heater.presets" command="SET_HEATER_TEMPERATURE" attribute-name="HEATER" ></toolInput>
                            </v-col>
                        </v-row>
                    </div>
                    <div v-for="(fan, index) in temperatureFans" v-bind:key="index+99" >
                        <v-divider class="my-2"></v-divider>
                        <v-row align="center">
                            <v-col class="flex-grow-0 py-2 pl-8 pr-0  colHeaterIcons">
                                <v-icon :color="(fan.target ? 'grey lighten-5' : 'grey darken-2')" :class="(fan.speed ? ' icon-rotate' : '')">mdi-fan</v-icon>
                            </v-col>
                            <v-col class="py-2 font-weight-bold"><span style="cursor: pointer;" @click="openHeater(fan)">{{ convertName(fan.name) }}</span></v-col>
                            <v-col class="py-2 text-center d-none d-sm-block"><small>{{ fan.target > 0 && fan.speed > 0 ? (fan.speed * 100).toFixed(0)+"%" : (fan.target > 0 ? "standby" : "off") }}</small></v-col>
                            <v-col class="py-2 text-center">
                                <span class="d-block">{{ fan.temperature ? fan.temperature.toFixed(1) : 0}}°C</span>
                                <span v-for="(values, key) of fan.additionValues" v-bind:key="key" class="d-block">{{ values.value.toFixed(1) }} {{ values.unit }}</span>
                            </v-col>
                            <v-col class="text-center py-2 pr-8 pr-0  vertical_align_center">
                                <toolInput :name="fan.name" :target="fan.target" command="SET_TEMPERATURE_FAN_TARGET" attribute-name="temperature_fan" :items="fan.presets" ></toolInput>
                            </v-col>
                        </v-row>
                    </div>
                    <div v-for="(sensor,index) in temperatureSensors" v-bind:key="index+999">
                        <v-divider class="my-2"></v-divider>
                        <v-row align="center">
                            <v-col class="flex-grow-0 py-2 pl-8 pr-0 colHeaterIcons">
                                <v-icon color="grey darken-2" :title="'min: '+sensor.min_temp+'° / max: '+sensor.max_temp+'°'">{{ sensor.icon }}</v-icon>
                            </v-col>
                            <v-col class="py-2 font-weight-bold">
                              <span style="cursor: pointer;" @click="openHeater(sensor)">{{ convertName(sensor.name) }}</span>
                            </v-col>
                            <v-col class="py-2 d-none d-sm-block"><span>&nbsp;</span></v-col>
                            <v-col class="py-2 text-center">
                              <v-tooltip top>
                                <template v-slot:activator="{ on, attrs }">
                                  <span
                                      style="cursor: default;"
                                      class="d-block px-0"
                                      v-bind="attrs"
                                      v-on="on"
                                  >{{ sensor.temperature ? sensor.temperature.toFixed(1) : 0 }}°C</span>
                                </template>
                                <span>min: {{ sensor.measured_min_temp ? sensor.measured_min_temp.toFixed(1) : 0}}°<br />max: {{ sensor.measured_max_temp ? sensor.measured_max_temp.toFixed(1) : 0 }}°</span>
                              </v-tooltip>
                              <span v-for="(values, key) of sensor.additionValues" v-bind:key="key" class="d-block">{{ values.value.toFixed(1) }} {{ values.unit }}</span>
                            </v-col>
                            <v-col class="text-center py-2 pr-8 vertical_align_center"><span>&nbsp;</span></v-col>
                        </v-row>
                    </div>
                    <v-divider class="my-2" v-if="boolTempchart"></v-divider>
                    <v-row v-if="boolTempchart">
                        <v-col class="px-8 pt-6">
                            <temp-chart ></temp-chart>
                        </v-col>
                    </v-row>
                </v-container>
            </v-card-text>
        </v-card>
        <v-dialog v-model="editHeater.bool" persistent :width="400">
            <v-card dark>
                <v-toolbar flat dense color="primary">
                    <v-toolbar-title>
                    <span class="subheading">
                        <v-icon left>mdi-{{ 'icon' in editHeater.object ? editHeater.object.icon : "info" }}</v-icon>
                        {{ 'name' in editHeater.object ? convertName(editHeater.object.name) : "UNKNOWN" }}
                    </span>
                    </v-toolbar-title>
                    <v-spacer></v-spacer>
                    <v-btn small class="minwidth-0" @click="editHeater.bool = false"><v-icon small>mdi-close-thick</v-icon></v-btn>
                </v-toolbar>
                <v-card-text class="pt-3">
                    <v-container class="px-0 py-0">
                        <v-row>
                            <v-col class="col-12">
                                <v-checkbox
                                    v-model="editHeater.boolTemperature"
                                    label="Show current temperature in chart"
                                    hide-details
                                    class="mt-0"
                                    v-if="'chartTemperature' in editHeater.object && editHeater.object.chartTemperature"
                                    @change="setVisibleTemperature"
                                ></v-checkbox>
                            </v-col>
                        </v-row>
                        <v-row>
                            <v-col class="col-12">
                                <v-checkbox
                                    v-model="editHeater.boolTarget"
                                    label="Show target temperature in chart"
                                    hide-details
                                    class="mt-0"
                                    v-if="'chartTarget' in editHeater.object && editHeater.object.chartTarget"
                                    @change="setVisibleTarget"
                                ></v-checkbox>
                            </v-col>
                        </v-row>
                        <v-row>
                            <v-col class="col-12">
                                <v-checkbox
                                    v-model="editHeater.boolPower"
                                    label="Show PWM-power in chart"
                                    hide-details
                                    class="mt-0"
                                    v-if="'chartPower' in editHeater.object && editHeater.object.chartPower"
                                    @change="setVisiblePower"
                                ></v-checkbox>
                            </v-col>
                        </v-row>
                        <v-row>
                            <v-col class="col-12 text-center">
                                <v-color-picker
                                    hide-canvas
                                    hide-mode-switch
                                    mode="hexa"
                                    v-model="editHeater.color"
                                    @input="setChartColor"
                                    class="mx-auto"
                                ></v-color-picker>
                            </v-col>
                        </v-row>
                    </v-container>
                </v-card-text>
            </v-card>
        </v-dialog>
    </div>
</template>

<script>
    import { mapState, mapGetters } from 'vuex'
    import toolInput from '../../inputs/ToolInput'
    import TempChart from '@/components/charts/TempChart'
    import {convertName} from "@/plugins/helpers"

    export default {
        components: {
            toolInput,
            TempChart,
        },
        data: function() {
            return {
                editHeater: {
                    bool: false,
                    object: {},
                    boolTemperature: false,
                    boolTarget: false,
                    boolPower: false,
                    color: "",
                }
            }
        },
        computed: {
            ...mapState({
                datasets: state => state.printer.tempHistory.datasets,
                boolTempchart: state => state.gui.dashboard.boolTempchart,
                printer: state => state.printer,
                cooldownGcode: state => state.gui.cooldownGcode,
                printer_state: state => state.printer.print_stats.state,
            }),
            ...mapGetters([
                'gui/getPreheatPresets'
            ]),
            heaters: {
                get () {
                    return this.$store.getters["printer/getHeaters"]
                }
            },
            temperatureFans: {
                get () {
                    return this.$store.getters["printer/getTemperatureFans"]
                }
            },
            temperatureSensors: {
                get () {
                    return this.$store.getters["printer/getTemperatureSensors"]
                }
            },
            boolTempchart: {
                get() {
                    return this.$store.state.gui.dashboard.boolTempchart
                },
                set: function(newVal) {
                    return this.$store.dispatch("gui/setSettings", { dashboard: { boolTempchart: newVal } })
                }
            },
            autoscaleTempchart: {
                get() {
                    return this.$store.state.gui.tempchart.autoscale
                },
                set: function(newVal) {
                    return this.$store.dispatch("gui/setSettings", { tempchart: { autoscale: newVal } })
                }
            }
        },
        created() {

        },
        methods: {
            convertName: convertName,
            cssBorderLeftColor(color) {
                return {
                  "border-left-color": color
                }
            },
            preheat(preset) {
                for (const [name, attributes] of Object.entries(preset.values)) {
                    if (attributes.bool) {
                        let gcode = "SET_HEATER_TEMPERATURE HEATER="+name+" TARGET="+attributes.value

                        if (attributes.type === "temperature_fan") {
                            const fanName = name.replace("temperature_fan ", "")
                            gcode = "SET_TEMPERATURE_FAN_TARGET temperature_fan="+fanName+" TARGET="+attributes.value
                        }

                        this.$store.commit('server/addEvent', { message: gcode, type: 'command' })
                        this.$socket.sendObj('printer.gcode.script', { script: gcode })
                    }
                }

                if (preset.gcode !== "") {
                    this.$store.commit('server/addEvent', { message: preset.gcode, type: 'command' })
                    this.$socket.sendObj('printer.gcode.script', { script: preset.gcode })
                }
            },
            cooldown() {
                this.$store.commit('server/addEvent', { message: this.cooldownGcode, type: 'command' })
                this.$socket.sendObj('printer.gcode.script', { script: this.cooldownGcode })
            },
            openHeater(object) {
                this.editHeater.object = object

                this.editHeater.color = object.chartColor
                this.editHeater.boolTemperature = 'chartTemperature' in object && object.chartTemperature !== undefined && 'visible' in object.chartTemperature ? object.chartTemperature.visible : false
                this.editHeater.boolTarget = 'chartTarget' in object && object.chartTarget !== undefined && 'visible' in object.chartTarget ? object.chartTarget.visible : false
                this.editHeater.boolPower = 'chartPower' in object && object.chartPower !== undefined && 'visible' in object.chartPower ? object.chartPower.visible : false

                this.editHeater.bool = true
            },
            setVisibleTemperature() {
                if ("name" in this.editHeater.object) {
                    this.$store.commit('printer/tempHistory/setVisible', { name: this.editHeater.object.name, type: 'temperature', value: this.editHeater.boolTemperature })
                    this.$store.dispatch('gui/setTempchartDatasetSetting', { name: this.editHeater.object.name, type: 'temperature', value: this.editHeater.boolTemperature })
                }
            },
            setVisibleTarget() {
                if ("name" in this.editHeater.object) {
                    this.$store.commit('printer/tempHistory/setVisible', { name: this.editHeater.object.name, type: 'target', value: this.editHeater.boolTarget })
                    this.$store.dispatch('gui/setTempchartDatasetSetting', { name: this.editHeater.object.name, type: 'target', value: this.editHeater.boolTarget })
                }
            },
            setVisiblePower() {
                if ("name" in this.editHeater.object) {
                    this.$store.commit('printer/tempHistory/setVisible', { name: this.editHeater.object.name, type: 'power', value: this.editHeater.boolPower })
                    this.$store.dispatch('gui/setTempchartDatasetSetting', { name: this.editHeater.object.name, type: 'power', value: this.editHeater.boolPower })
                }
            },
            setChartColor(value) {
                if ("name" in this.editHeater.object) {
                    this.$store.commit('printer/tempHistory/setColor', { name: this.editHeater.object.name, value: value })
                    this.$store.dispatch('gui/setTempchartDatasetSetting', { name: this.editHeater.object.name, type: 'color', value: value })
                }
            }
        },
    }
</script>