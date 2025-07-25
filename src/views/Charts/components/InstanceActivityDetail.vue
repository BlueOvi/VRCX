<template>
    <div style="width: 100%">
        <div style="height: 25px; margin-top: 60px">
            <transition name="el-fade-in-linear">
                <Location
                    v-show="!isLoading"
                    class="location"
                    :location="activityDetailData[0]?.location"
                    is-open-previous-instance-info-dialog />
            </transition>
        </div>

        <div ref="activityDetailChartRef"></div>
    </div>
</template>

<script setup>
    import { ref, watch, computed, onDeactivated, onMounted } from 'vue';
    import dayjs from 'dayjs';
    import { storeToRefs } from 'pinia';

    import { loadEcharts, timeToText } from '../../../shared/utils';
    import { useUserStore, useAppearanceSettingsStore } from '../../../stores';

    const { isDarkMode, dtHour12 } = storeToRefs(useAppearanceSettingsStore());
    const { showUserDialog } = useUserStore();
    const { currentUser } = storeToRefs(useUserStore());

    const props = defineProps({
        activityDetailData: {
            type: Array,
            required: true
        },
        barWidth: {
            type: Number,
            required: true,
            default: 10
        }
    });

    const activityDetailChartRef = ref(null);

    const echarts = ref(null);
    const isLoading = ref(true);
    const echartsInstance = ref(null);
    const usersFirstActivity = ref(null);
    const resizeObserver = ref(null);

    const startTimeStamp = computed(() => {
        return props.activityDetailData.find((item) => item.user_id === currentUser.value.id)?.joinTime.valueOf();
    });

    const endTimeStamp = computed(() => {
        return props.activityDetailData.find((item) => item.user_id === currentUser.value.id)?.leaveTime.valueOf();
    });

    watch(
        () => isDarkMode.value,
        () => {
            if (echartsInstance.value) {
                echartsInstance.value.dispose();
                echartsInstance.value = null;
                initEcharts();
            }
        }
    );

    watch(
        () => dtHour12.value,
        () => {
            if (echartsInstance.value) {
                initEcharts();
            }
        }
    );

    initResizeObserver();

    onMounted(() => {
        initEcharts(true);
    });

    onDeactivated(() => {
        // prevent switch tab play resize animation
        resizeObserver.value.disconnect();
    });

    function initResizeObserver() {
        resizeObserver.value = new ResizeObserver((entries) => {
            for (const entry of entries) {
                echartsInstance.value.resize({
                    width: entry.contentRect.width,
                    animation: {
                        duration: 300
                    }
                });
            }
        });
    }

    async function initEcharts(isFirstLoad = false) {
        if (!echarts.value) {
            echarts.value = await loadEcharts();
        }

        const chartsHeight = props.activityDetailData.length * (props.barWidth + 10) + 200;
        const chartDom = activityDetailChartRef.value;
        if (!echartsInstance.value) {
            echartsInstance.value = echarts.value.init(chartDom, `${isDarkMode.value ? 'dark' : null}`, {
                height: chartsHeight,
                useDirtyRect: props.activityDetailData.length > 80
            });
            resizeObserver.value.observe(chartDom);
        }

        echartsInstance.value.resize({
            height: chartsHeight,
            animation: {
                duration: 300
            }
        });

        echartsInstance.value.setOption(isFirstLoad ? {} : getNewOption(), { lazyUpdate: true });
        echartsInstance.value.on('click', 'yAxis', handleClickYAxisLabel);

        setTimeout(() => {
            isLoading.value = false;
        }, 200);
    }

    function handleClickYAxisLabel(params) {
        const userData = usersFirstActivity.value[params.dataIndex];
        if (userData?.user_id) {
            showUserDialog(userData.user_id);
        }
    }

    function getNewOption() {
        // grouping player activity entries by user_id and calculate below:
        // 1. offset: the time from startTimeStamp or the previous entry's tail to the current entry's joinTime
        // 2. time: the time the user spent in the instance
        // 3. tail: the time from startTimeStamp to the current entry's leaveTime
        // 4. entry: the original activity detail entry
        const userGroupedEntries = new Map();
        // uniqueUserEntries has each user's first entry and used to keep the order of the users calculated in InstanceActivity.vue
        const uniqueUserEntries = [];
        for (const entry of props.activityDetailData) {
            if (!userGroupedEntries.has(entry.user_id)) {
                userGroupedEntries.set(entry.user_id, []);
                uniqueUserEntries.push(entry);
            }
            const elements = userGroupedEntries.get(entry.user_id);
            const offset = Math.max(
                0,
                elements.length === 0
                    ? entry.joinTime.valueOf() - startTimeStamp.value
                    : entry.joinTime.valueOf() - startTimeStamp.value - elements[elements.length - 1].tail
            );
            const tail =
                elements.length === 0 ? offset + entry.time : elements[elements.length - 1].tail + offset + entry.time;
            const element = { offset, time: entry.time, tail, entry };
            elements.push(element);
        }
        usersFirstActivity.value = uniqueUserEntries;

        const generateSeries = () => {
            const maxEntryCount = Math.max(...Array.from(userGroupedEntries.values()).map((entries) => entries.length));
            const placeholderSeries = (data) => {
                return {
                    name: 'Placeholder',
                    type: 'bar',
                    stack: 'Total',
                    itemStyle: {
                        borderColor: 'transparent',
                        color: 'transparent'
                    },
                    emphasis: {
                        itemStyle: {
                            borderColor: 'transparent',
                            color: 'transparent'
                        }
                    },
                    data
                };
            };
            const timeSeries = (data) => {
                return {
                    name: 'Time',
                    type: 'bar',
                    stack: 'Total',
                    colorBy: 'data',
                    barWidth: props.barWidth,
                    emphasis: {
                        focus: 'self'
                    },
                    itemStyle: {
                        borderRadius: 2,
                        shadowBlur: 2,
                        shadowOffsetX: 0.7,
                        shadowOffsetY: 0.5
                    },
                    data
                };
            };

            // generate series having placeholder and time series for each user
            const series = Array(maxEntryCount)
                .fill(0)
                .flatMap((_, index) => {
                    return [
                        placeholderSeries(
                            uniqueUserEntries.map((entry) => {
                                const element = userGroupedEntries.get(entry.user_id)[index];
                                return element ? element.offset : 0;
                            })
                        ),
                        timeSeries(
                            uniqueUserEntries.map((entry) => {
                                const element = userGroupedEntries.get(entry.user_id)[index];
                                return element ? element.time : 0;
                            })
                        )
                    ];
                });

            return series;
        };

        const friendOrFavIcon = (display_name) => {
            const foundItem = props.activityDetailData.find((item) => item.display_name === display_name);

            if (!foundItem) {
                return '';
            }

            if (foundItem.isFavorite) {
                return '⭐';
            }
            if (foundItem.isFriend) {
                return '💚';
            }
            return '';
        };

        const getTooltip = (params) => {
            const activityDetailData = props.activityDetailData;
            const param = params;
            const userData = uniqueUserEntries[param.dataIndex];
            const isTimeSeries = params.seriesIndex % 2 === 1;
            if (!isTimeSeries) {
                return '';
            }
            const targetEntryIndex = Math.floor(params.seriesIndex / 2);

            if (!activityDetailData || !userData) {
                return '';
            }

            // first, find the user's entries, then get the focused entry
            const instanceData = userGroupedEntries.get(userData.user_id)[targetEntryIndex].entry;

            const format = dtHour12.value ? 'hh:mm:ss A' : 'HH:mm:ss';
            const formattedLeftDateTime = dayjs(instanceData.leaveTime).format(format);
            const formattedJoinDateTime = dayjs(instanceData.joinTime).format(format);

            const timeString = timeToText(instanceData.time, true);
            const color = param.color;

            return `
                    <div style="display: flex; align-items: center;">
                        <div style="width: 10px; height: 55px; background-color: ${color}; margin-right: 5px;"></div>
                        <div>
                            <div>${instanceData.display_name} ${friendOrFavIcon(instanceData.display_name)}</div>
                            <div>${formattedJoinDateTime} - ${formattedLeftDateTime}</div>
                            <div>${timeString}</div>
                        </div>
                    </div>
                    `;
        };

        const format = dtHour12.value ? 'hh:mm A' : 'HH:mm';

        const echartsOption = {
            tooltip: {
                trigger: 'item',
                axisPointer: {
                    type: 'shadow'
                },
                formatter: getTooltip
            },
            grid: {
                top: 50,
                left: 160,
                right: 90
            },
            yAxis: {
                type: 'category',
                axisLabel: {
                    interval: 0,
                    formatter: (value) => {
                        const MAX_LENGTH = 20;
                        const len = value.length;
                        return `${friendOrFavIcon(value)} ${len > MAX_LENGTH ? `${value.substring(0, MAX_LENGTH)}...` : value}`;
                    }
                },
                inverse: true,
                data: uniqueUserEntries.map((item) => item.display_name),
                triggerEvent: true
            },
            xAxis: {
                type: 'value',
                min: 0,
                max: endTimeStamp.value - startTimeStamp.value,
                axisLine: { show: true },
                axisLabel: {
                    formatter: (value) => dayjs(value + startTimeStamp.value).format(format)
                },
                splitLine: { lineStyle: { type: 'dashed' } }
            },
            series: generateSeries(),
            backgroundColor: 'rgba(0, 0, 0, 0)'
        };

        return echartsOption;
    }

    defineExpose({
        echartsInstance,
        initEcharts
    });
</script>

<style scoped>
    .location {
        display: flex;
        align-items: center;
        justify-content: center;
    }
</style>
