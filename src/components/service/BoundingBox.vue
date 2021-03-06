<script>
import bem from 'easy-bem';
import classnames from 'classnames';
import { isEmpty, replacedProp } from '../../core';
import { directionNames } from '../../core/utils';
import { ResizeEvent } from '../../core/events';
import { SimpleHandler } from '../handlers';

const cn = bem('vue-bounding-box');

const HORIZONTAL_DIRECTIONS = ['east', 'west', null];
const VERTICAL_DIRECTIONS = ['south', 'north', null];

export default {
	name: 'BoundingBox',
	props: {
		handlers: {
			type: Object,
			default() {
				return {
					eastNorth: true,
					north: true,
					westNorth: true,
					west: true,
					westSouth: true,
					south: true,
					eastSouth: true,
					east: true,
				};
			},
		},
		handlerComponent: {
			type: [Object, String],
			default() {
				return SimpleHandler;
			},
		},
		handlersClasses: {
			type: Object,
			default() {
				return {};
			},
		},
		lines: {
			type: Object,
			default() {
				return {
					west: true,
					north: true,
					east: true,
					south: true,
				};
			},
		},
		lineComponent: {
			type: [Object, String],
		},
		linesClasses: {
			type: Object,
			default() {
				return {};
			},
		},
		scalable: {
			type: Boolean,
			default: true
		},
		// Deprecated props:
		classname: {
			type: String,
			validator(value) {
				return replacedProp(value, 'classname', 'class');
			}
		},
		linesClassnames: {
			type: Object,
			default() {
				return {};
			},
			validator(value) {
				return replacedProp(value, 'linesClassnames', 'linesClasses');
			}
		},
		handlersClassnames: {
			type: Object,
			default() {
				return {};
			},
			validator(value) {
				return replacedProp(value, 'handlersClassnames', 'handlersClasses');
			}
		},
	},
	data() {
		const points = [];
		HORIZONTAL_DIRECTIONS.forEach(hDirection => {
			VERTICAL_DIRECTIONS.forEach(vDirection => {
				if (hDirection !== vDirection) {
					let { name, classname, } = directionNames(hDirection, vDirection);
					points.push({
						name,
						classname,
						verticalDirection: vDirection,
						horizontalDirection: hDirection,
					});
				}
			});
		});
		return {
			points,
		};
	},
	computed: {
		classes() {
			const handlers = isEmpty(this.handlersClasses) ? this.handlersClassnames : this.handlersClasses;
			const lines = isEmpty(this.linesClasses) ? this.linesClassnames : this.linesClasses;

			return {
				root: classnames(
					cn(),
					this.classname
				),
				handlers,
				lines,
			};
		},
		lineNodes() {
			const lines = [];
			this.points.forEach(point => {
				if ((!point.horizontalDirection || !point.verticalDirection) && this.lines[point.name]) {
					lines.push({
						name: point.name,
						component: this.lineComponent,
						class: classnames(
							this.classes.lines.default,
							this.classes.lines[point.name],
							!this.scalable && this.classes.lines.disabled
						),
						hoverClass: this.classes.lines.hover,
						verticalDirection: point.verticalDirection,
						horizontalDirection: point.horizontalDirection,
						disabled: !this.scalable
					});
				}
			});
			return lines;
		},
		handlerNodes() {
			const handlers = [];
			this.points.forEach(point => {
				if (this.handlers[point.name]) {
					handlers.push({
						name: point.name,
						component: this.handlerComponent,
						class: classnames(
							this.classes.handlers.default,
							this.classes.handlers[point.name]
						),
						hoverClass: this.classes.handlers.hover,
						verticalDirection: point.verticalDirection,
						horizontalDirection: point.horizontalDirection,
						disabled: !this.scalable
					});
				}
			});
			return handlers;
		},
	},
	beforeMount() {
		window.addEventListener('mouseup', this.onMouseUp, { passive: false, });
		window.addEventListener('mousemove', this.onMouseMove, { passive: false, });
		window.addEventListener('touchmove', this.onTouchMove, { passive: false, });
		window.addEventListener('touchend', this.onTouchEnd, { passive: false, });
	},
	beforeDestroy() {
		window.removeEventListener('mouseup', this.onMouseUp);
		window.removeEventListener('mousemove', this.onMouseMove);
		window.removeEventListener('touchmove', this.onTouchMove);
		window.removeEventListener('touchend', this.onTouchEnd);
	},
	mounted() {
		this.touches = [];
	},
	methods: {
		onHandlerDrag(dragEvent, horizontalDirection, verticalDirection) {
			const { left, top, } = dragEvent.shift();

			const directions = {
				left: 0,
				right: 0,
				top: 0,
				bottom: 0,
			};

			if (horizontalDirection === 'west') {
				directions.left -= left;
			} else if (horizontalDirection === 'east') {
				directions.right += left;
			}
			if (verticalDirection === 'north') {
				directions.top -= top;
			} else if (verticalDirection === 'south') {
				directions.bottom += top;
			}

			let respectDirection;
			if (!verticalDirection && horizontalDirection) {
				respectDirection = 'width';
			} else if (verticalDirection && !horizontalDirection) {
				respectDirection = 'height';
			}

			if (this.scalable) {
				this.$emit('resize', new ResizeEvent(
					directions,
					{
						allowedDirections: {
							left: horizontalDirection === 'west' || !horizontalDirection,
							right: horizontalDirection === 'east' || !horizontalDirection,
							bottom: verticalDirection === 'south' || !verticalDirection,
							top: verticalDirection === 'north' || !verticalDirection,
						},
						preserveAspectRatio: dragEvent.nativeEvent && dragEvent.nativeEvent.shiftKey,
						respectDirection,
					}
				));
			}
		},
	},
};
</script>

<template>
  <div
    ref="box"
    :class="classes.root"
  >
    <slot />
    <div>
      <component
        :is="line.component"
        v-for="line in lineNodes"
        :key="line.name"
        :classname="line.class"
        :hover-classname="line.hoverClass"
        :position="line.name"
        :disabled="line.disabled"
        @drag="onHandlerDrag($event, line.horizontalDirection, line.verticalDirection)"
      />
    </div>
    <div>
      <component
        :is="handler.component"
        v-for="handler in handlerNodes"
        :key="handler.name"
        :classname="handler.class"
        :hover-classname="handler.hoverClass"
        :horizontal-position="handler.horizontalDirection"
        :vertical-position="handler.verticalDirection"
        :disabled="handler.disabled"
        @drag="onHandlerDrag($event, handler.horizontalDirection, handler.verticalDirection)"
      />
    </div>
  </div>
</template>

<style lang="scss">
.vue-bounding-box {
  position: relative;
  height: 100%;
  width: 100%;
}
</style>
