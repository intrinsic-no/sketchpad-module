<template>
    <svg
        ref="canvas"
        class="fit intrinsic-sketchpad block"
        @mousedown="handlePointerDown"
        @mousemove="handlePointerMove"
        @mouseup="handlePointerUp"
        @mouseleave="handlePointerLeave"
        @touchstart="handleTouchDown"
        @touchmove="handleTouchMove"
        @touchend="handleTouchUp"
        @touchcancel="handleTouchUp"
    >
        <path
            v-for="(path, index) in pathsWithData"
            :key="`path${index}`"
            :d="path.path"
            :fill="path.color"
        />

        <path
            :d="pathData"
            :fill="color"
        />
    </svg>
</template>

<script>
import { getStroke } from 'perfect-freehand';
import { Canvg } from 'canvg';
import { getSvgPathFromStroke } from '@intrinsic-no/sketchpad/src/utils';

export default {
    props: {
        readonly: {
            type: Boolean,
            default() {
                return false;
            },
        },
        color: {
            type: String,
            default() {
                return '#000';
            },
        },
        options: {
            type: Object,
            default() {
                return {};
            },
        },
        width: {
            type: Number,
            default: 100,
        },
        height: {
            type: Number,
            default: 100,
        },
        modelValue: {
            type: Array,
            default() {
                return [];
            },
        },
    },
    emits: ['update:modelValue'],

    data() {
        return {
            points: [],
            paths: this.modelValue,
        };
    },

    computed: {
        allOptions() {
            return {
                size: 3,
                thinning: 0.33,
                smoothing: 0.2,
                streamline: 0.5,
                easing: t => t,
                start: {
                    taper: 0,
                    easing: t => t,
                    cap: true,
                },
                end: {
                    taper: 0,
                    easing: t => t,
                    cap: true,
                },
                ...this.options,
            };
        },

        stroke() {
            return getStroke(this.points, this.allOptions);
        },

        pathData() {
            return getSvgPathFromStroke(this.stroke);
        },

        pathsWithData() {
            return JSON.parse(JSON.stringify(this.paths)).map(points => {
                points.path = getSvgPathFromStroke(getStroke(points.path, this.allOptions));

                return points;
            });
        },
    },

    watch: {
        paths: {
            deep: true,
            handler(value) {
                this.$emit('update:modelValue', value);
            },
        },

        modelValue: {
            deep: true,
            handler(value) {
                this.paths = value;
            },
        },
    },

    methods: {
        getCanvasX(e, type = 'mouse') {
            let moveX = e.pageX;

            if (type === 'touch') {
                moveX = e.touches[0].clientX;
            }

            if (!this.$refs.canvas) {
                return moveX;
            }

            return moveX - this.$refs.canvas.getBoundingClientRect().left - window.scrollX;
        },

        getCanvasY(e, type = 'mouse') {
            let moveY = e.pageY;

            if (type === 'touch') {
                moveY = e.touches[0].clientY;
            }

            return moveY - this.$refs.canvas.getBoundingClientRect().top - window.scrollY;
        },

        handlePointerDown(e) {
            if (this.readonly) {
                return;
            }

            this.points = [[this.getCanvasX(e), this.getCanvasY(e), e.pressure]];
        },

        handlePointerMove(e) {
            if (this.readonly) {
                return;
            }

            if (e.buttons !== 1) {
                return;
            }

            this.points = [...this.points, [this.getCanvasX(e), this.getCanvasY(e), e.pressure]];
        },

        handlePointerUp() {
            if (this.readonly) {
                return;
            }
            const path = JSON.parse(JSON.stringify(this.points));

            this.paths.push({
                path,
                color: this.color.toString(),
            });
            this.points = [];
        },

        handlePointerLeave(e) {
            if (e.buttons !== 1) {
                return;
            }

            this.handlePointerUp();
        },

        handleTouchDown(e) {
            if (this.readonly) {
                return;
            }

            e.preventDefault();

            this.points = [[this.getCanvasX(e, 'touch'), this.getCanvasY(e, 'touch'), e.pressure]];
        },

        handleTouchMove(e) {
            if (this.readonly) {
                return;
            }

            e.preventDefault();

            this.points = [
                ...this.points,
                [this.getCanvasX(e, 'touch'), this.getCanvasY(e, 'touch'), e.pressure],
            ];
        },

        handleTouchUp(e) {
            if (this.readonly) {
                return;
            }

            e.preventDefault();

            const path = JSON.parse(JSON.stringify(this.points));

            this.paths.push({
                path,
                color: this.color.toString(),
            });
            this.points = [];
        },

        clear() {
            if (this.readonly) {
                return;
            }

            this.paths = [];
        },

        undo() {
            if (this.readonly) {
                return;
            }

            this.paths.pop();
        },

        async export() {
            const canvas = document.createElement('canvas');
            const ctx = canvas.getContext('2d');

            canvas.width = this.width;
            canvas.height = this.height;

            const item = await Canvg.fromString(ctx, this.$refs.canvas.outerHTML);

            item.start();
            return canvas.toDataURL();
        },
    },
};
</script>

<style lang="sass" scoped>
.intrinsic-sketchpad
    touch-action: none
    user-select: none
    -webkit-user-select: none
    -webkit-touch-callout: none
</style>
