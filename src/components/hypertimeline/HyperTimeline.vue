<template>
  <!-- <div id="time-line" @wheel="wheelZoom" class=""> -->
  <div id="time-line" class="">
    
    <div id="controls">
        RATIO: {{ratio}} px : 1 year
        <button @click="decrement">-</button>
        <button @click="increment">+</button>
    </div>

    <div id="upper-timeline" :style="{ width: totalWidthInPixels + 'px' }">
        <div class="lane" v-for="(timeframe, index) in timeFrames" :key="index">
            <div class="intervals-container" :style="{ marginLeft: cv(timeFrames[index].events[0].from.year) + 'px' }">
                <interval 
                    v-for="(interval, index) in timeframe.events"
                    :key="index"
                    :data="interval"               
                    :ratio="ratio"/>
            </div>
        </div>
    </div>
    
    <timeframe :from="from" :to="to" :vw="viewWidthInYears" :ratio="ratio" />

    <div id="lower-timeline" :style="{ width: totalWidthInPixels + 'px', height: 2000 + 'px' }">
        <div class="lane" v-for="(lane, index) in intervalScheduling" :key="index">
            <div class="intervals-container" :style="{ marginLeft: cv(lane[0].from.year) + 'px' }">
                <interval 
                    v-for="(interval, index) in lane"
                    :key="index"
                    :data="interval"
                    :left="index > 0 ? lane[index - 1].to.year : 0"
                    :ratio="ratio"/>
            </div>
        </div>
    </div>

  </div>
</template>

<script>
/* Imports */
import Vue from 'vue'
import _Interval from '../../components/hypertimeline/Interval'
import Timeframe from '../../components/hypertimeline/Timeframe'

import {upper, lower} from './Data'
import { Timeline } from './Helper'

/* Constants */
const defaultZoomSensitivity = 0.0001
const DEFAULT_POSITION = 0 // year 0 ~ Jesus is born!
const DEFAULT_FROM = -4000 // 4,000 BC
const PRESENT = new Date().getFullYear()

/**
 * Merges recursively source:(Timeline|interval|mark) into res
 * @return res:(interval|mark)[]
 */
let mergeRecursive = (source) => {
    let res = []
    if (source instanceof Timeline) {
        source.events.forEach((event) => {
            res = res.concat(
                mergeRecursive(event)
            )
        })
    } else {
        // is either a single interval (with possible sub-interbals) or mark
        return [source]
    }

    return res
}

export default Vue.extend({
    // html-args
    props: ['initialZoom', 'initialPosition'],
    components: {
        'interval': _Interval,
        'timeframe': Timeframe,
    },
    data() {
        return {
            from: DEFAULT_FROM,
            to: PRESENT + 1,
            vw: 1, // view width
            zoom: this.initialZoom,
            zoomSensitivity: defaultZoomSensitivity,
            position: (this.initialPosition) ? this.initialPosition : DEFAULT_POSITION,
            // todo: merge these two together
            timeFrames: upper,
            events: lower.events,
        }
    },
    mounted() {
        let el = document.getElementById('time-line')
        this.vw = el.offsetWidth
        el.scrollLeft = this.cv(this.position, this.from, this.ratio) - (this.vw / 2)
    },
    methods: {
        increment() {
            this.zoom++
        },
        decrement() {
            // if (this.zoom > 1) {
                this.zoom--
            // }
        },
        wheelZoom(event) {
            let dY = event.deltaY
            this.zoom += dY * -1 * this.zoomSensitivity
        },
        // year to pixels, from left.
        // 0px = -4,000 BC
        cv(from) {
            let left = this.from
            return Math.abs(left - from) * this.ratio
        }
    },
    computed: {
        roundedZoom() {
            return Number((this.zoom).toFixed(1))
        },
        ratio() {
            /*
            zoom=2 -> ratio = 4px/1yr ~ 4
            zoom=1 -> ratio = 2px/1yr ~ 2
            zoom=0 -> ratio  = 1px/1yr ~ 1
            zoom=-1 -> ratio = 1px/2yr ~ 0.5
            zoom=-2 -> ratio = 1px/4yr ~ 0.25
            zoom=-3 -> ratio = 1px/8yr ~ 0.125
            then ratio=(1px/2^-zoom yr)
            */
            let r = 1 / Math.pow(2, -this.zoom)
            return r
        },
        totalWidthInPixels() {
            return (this.to - this.from) * this.ratio
        },
        viewWidthInYears() {
            let vwiy = this.vw / this.ratio
            // console.log('view width in years', vwiy)
            return vwiy
        },
        intervalScheduling() {
            // first merge it all in a single overlaped lane
            let merged = mergeRecursive(lower)

            // then sort them by "finish time" asc (`event.to` in this case)
            let sorted = merged.sort((a, b) => {
                return a.to.year - b.to.year
            })

            let lanes = [] // array of (array of compatible event)
            // distribute `sorted` in the least amount of non overlaping lanes
            sorted.forEach(event => {
                let i = 0
                for (; i < lanes.length; i++) {
                    let n = lanes[i].length - 1
                    let last = lanes[i][n].to.year
                    let laneIsCompatible = last <= event.from.year
                    if (laneIsCompatible) {
                        break
                    }
                }
                let newLaneNeeded = lanes.length === i
                if (newLaneNeeded) {
                    lanes[i] = []
                }
                lanes[i].push(event)
            })

            return lanes
        }
    }
})
</script>

<style lang="scss">
#time-line {
	background: hsl(39, 30%, 85%);
    width: 75%;
    position: relative;
    overflow-y: hidden;
    overflow-x: scroll;

}

#upper-timeline {
    background: hsl(39, 30%, 85%);
}

#lower-timeline {
	background: hsl(39, 30%, 85%);
}

#controls {
    position: fixed;
    bottom: 0%;
    right: 25%;
    margin: 0 35px 20px 0;
}

.row {
    display: inline-flex;
}

.guide {
    &:before {
        content: "";
        height: 100%;
        width: 2px;
        left: calc(50% - 1px);
        background: red;
        position: absolute;
    }
}

.container {
    width: 4000px;
    display:inline-block;
    background: yellow;
}

.lane {
    display: flex;
    flex-direction: row;
    background: none;
    width: 100%;

    .intervals-container {
        display: flex;
    }
}

/* for dev/debugging purposes */
.debugging {
    
    .lane {
        padding: 2px;
        margin: 2px 0;
        border-bottom: 2px dashed white;
    }

    .intervals-container {
        background: #00ff08;
        padding: 10px;
    }

    #upper-timeline{
        background: blue;
    }

    #lower-timeline{
        background: purple;
    }
}
</style>
