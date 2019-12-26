<template>
    <div
        class="dnd-grid-container"
        :style="style"
    >
        <slot></slot>
        <box
            class="placeholder"
            box-id="::placeholder::"
        ></box>
    </div>
</template>

<style>
    .dnd-grid-container {
        position: relative;
        transition: min-width ease-out 0.1s, min-height ease-out 0.1s;
    }

    .dnd-grid-container .dnd-grid-box.placeholder {
        border: 1px dashed #000;
        background: none;
        z-index: 0;
    }

    .dnd-grid-container .dnd-grid-box.placeholder {
        transition: none !important;
    }
</style>

<script>
    import Box from './Box'
    import * as utils from './utils'

    export const List = new Set()

    export default {
        name: 'DndGridContainer',
        components: {
            Box
        },
        props: {
            layout: {
                type: Array,
                required: true
            },
            cell-size: {
                type: Object,
                default () {
                    return {
                        w: 100,
                        h: 100
                    }
                }
            },
            max-column-count: {
                type: Number,
                default: Infinity
            },
            max-row-count: {
                type: Number,
                default: Infinity
            },
            margin: {
                type: Number,
                default: 5
            },
            outerMargin: {
                type: Number,
                default: 0
            },
            bubbleUp: {
                type: Boolean,
                default: false
            },
            auto-add-layout-for-new-box: {
                type: Boolean,
                required: false,
                default: true
            },
            default-size: {
                type: Object,
                required: false,
                default () {
                    return {
                        w: 1,
                        h: 1
                    }
                }
            },
            fix-layout-on-load: {
                type: Boolean,
                required: false,
                default: true
            }
        },
        watch: {
            layout (newLayout) {
                if (this.fix-layout-on-load) {
                    if (utils.layoutHasCollisions(newLayout)) {
                        this.updateLayout(utils.fixLayout(newLayout, this.bubbleUp))
                    }
                }
            }
        },
        data () {
            return {
                placeholder: {
                    hidden: true,
                    position: {
                        x: 0,
                        y: 0,
                        w: 1,
                        h: 1
                    }
                },
                dragging: {
                    boxLayout: null,
                    offset: {
                        x: 0,
                        y: 0
                    }
                },
                resizing: {
                    boxLayout: null,
                    offset: {
                        x: 0,
                        y: 0
                    }
                },
                isMounted: false
            }
        },
        computed: {
            style () {
                var layoutSize = utils.getLayoutSize(this.layout)
                return {
                    minWidth: (
                        (layoutSize.w * this.cell-size.w) +
                        ((layoutSize.w - 1) * this.margin) +
                        (2 * this.outerMargin)
                    ) + 'px',
                    minHeight: (
                        (layoutSize.h * this.cell-size.h) +
                        ((layoutSize.h - 1) * this.margin) +
                        (2 * this.outerMargin)
                    ) + 'px'
                }
            },
            pinnedLayout () {
                return this.layout.filter((boxLayout) => {
                    return boxLayout.pinned
                })
            },
            layoutMap () {
                let map = new Map()
                this.layout.forEach(boxLayout => {
                    map.set(boxLayout.id, boxLayout)
                })
                return map
            }
        },
        methods: {
            getBoxLayoutById (id) {
                if (id === '::placeholder::') {
                    return this.placeholder
                }
                return this.layoutMap.get(id)
            },
            getPixelPositionById (id) {
                if (this.dragging.boxLayout && this.dragging.boxLayout.id === id) {
                    let pixels = utils.positionToPixels(this.dragging.boxLayout.position, this.cell-size, this.margin, this.outerMargin)
                    pixels.x += this.dragging.offset.x
                    pixels.y += this.dragging.offset.y
                    return pixels
                }

                if (this.resizing.boxLayout && this.resizing.boxLayout.id === id) {
                    let pixels = utils.positionToPixels(this.resizing.boxLayout.position, this.cell-size, this.margin, this.outerMargin)
                    pixels.w += this.resizing.offset.x
                    pixels.h += this.resizing.offset.y
                    return pixels
                }

                var boxLayout = this.getBoxLayoutById(id)
                return utils.positionToPixels(boxLayout.position, this.cell-size, this.margin, this.outerMargin)
            },
            isBoxVisible (id) {
                var boxLayout = this.getBoxLayoutById(id)
                return boxLayout && !boxLayout.hidden
            },
            getPositionByPixel (x, y) {
                return {
                    x: Math.round(x / (this.cell-size.w + this.margin)),
                    y: Math.round(y / (this.cell-size.h + this.margin))
                }
            },
            updateLayout (layout) {
                this.$emit('update:layout', layout)
            },
            registerBox (box) {
                this.enableResizing(box)
                this.enableDragging(box)
                if (this.isMounted && this.auto-add-layout-for-new-box) {
                    this.createBoxLayout(box.$props.box-id)
                }
            },
            unregisterBox (box) {
            },
            enableDragging (box) {
                var initialLayout
                var isDragging = false

                let validateTargetPosition = (targetX, targetY) => {
                    if (targetX + this.dragging.boxLayout.position.w > this.max-column-count) {
                        targetX = this.max-column-count - this.dragging.boxLayout.position.w
                    } else {
                        if (targetX < 0) {
                            targetX = 0
                        }
                    }
                    if (targetY + this.dragging.boxLayout.position.h > this.max-row-count) {
                        targetY = this.max-row-count - this.dragging.boxLayout.position.h
                    } else {
                        if (targetY < 0) {
                            targetY = 0
                        }
                    }
                    return {
                        targetX,
                        targetY
                    }
                }

                box.$on('dragStart', evt => {
                    var boxLayout = this.getBoxLayoutById(box.box-id)
                    if (boxLayout.pinned) {
                        return
                    }
                    isDragging = true

                    // find box
                    this.dragging.boxLayout = boxLayout
                    this.placeholder = {
                        ...this.dragging.boxLayout
                    }

                    // clone layout
                    initialLayout = utils.sortLayout(this.layout)

                    this.$emit('drag:start', initialLayout)
                })

                box.$on('dragUpdate', evt => {
                    if (!isDragging) {
                        return
                    }
                    this.dragging.offset.x = evt.offset.x
                    this.dragging.offset.y = evt.offset.y

                    var moveBy = this.getPositionByPixel(evt.offset.x, evt.offset.y)
                    if (!utils.isFree(this.pinnedLayout, {
                        ...this.dragging.boxLayout.position,
                        x: this.dragging.boxLayout.position.x + moveBy.x,
                        y: this.dragging.boxLayout.position.y + moveBy.y
                    })) {
                        return
                    }

                    let { targetX, targetY } = validateTargetPosition(
                        this.dragging.boxLayout.position.x + moveBy.x,
                        this.dragging.boxLayout.position.y + moveBy.y
                    )

                    // check if box has moved
                    if (this.placeholder.position.x === targetX && this.placeholder.position.y === targetY) {
                        return
                    }
                    this.placeholder = utils.updateBoxPosition(this.placeholder, {
                        x: targetX,
                        y: targetY
                    })

                    var newLayout = [ this.placeholder ]
                    initialLayout.forEach((boxLayout) => {
                        if (boxLayout.id === this.dragging.boxLayout.id) {
                            return
                        }
                        newLayout.push(utils.moveBoxToFreePlace(newLayout, boxLayout, this.bubbleUp))
                    })

                    if (this.bubbleUp) {
                        newLayout = utils.layoutBubbleUp(newLayout)
                        this.placeholder = newLayout.find((boxLayout) => {
                            return boxLayout.id === this.dragging.boxLayout.id
                        })
                    }
                    this.updateLayout(newLayout)

                    this.$emit('drag:update', newLayout)
                })

                box.$on('dragEnd', evt => {
                    if (!isDragging) {
                        return
                    }
                    var moveBy = this.getPositionByPixel(evt.offset.x, evt.offset.y)
                    if (utils.isFree(this.pinnedLayout, {
                        ...this.dragging.boxLayout.position,
                        x: this.dragging.boxLayout.position.x + moveBy.x,
                        y: this.dragging.boxLayout.position.y + moveBy.y
                    })) {
                        let { targetX, targetY } = validateTargetPosition(
                            this.dragging.boxLayout.position.x + moveBy.x,
                            this.dragging.boxLayout.position.y + moveBy.y
                        )

                        this.placeholder = utils.updateBoxPosition(this.placeholder, {
                            x: targetX,
                            y: targetY
                        })
                    }

                    this.dragging.boxLayout = utils.updateBoxPosition(this.dragging.boxLayout, {
                        x: this.placeholder.position.x,
                        y: this.placeholder.position.y
                    })

                    var newLayout = [ this.dragging.boxLayout ]
                    initialLayout.forEach((boxPosition) => {
                        if (boxPosition.id === this.dragging.boxLayout.id) {
                            return
                        }
                        newLayout.push(utils.moveBoxToFreePlace(newLayout, boxPosition, this.bubbleUp))
                    })

                    if (this.bubbleUp) {
                        newLayout = utils.layoutBubbleUp(newLayout)
                    }
                    this.updateLayout(newLayout)

                    this.dragging.boxLayout = null
                    this.dragging.offset.x = 0
                    this.dragging.offset.y = 0

                    this.placeholder.hidden = true
                    isDragging = false

                    this.$emit('drag:end', newLayout)
                })
            },
            enableResizing (box) {
                var initialLayout
                var isResizing = false

                let validateTargetSize = (targetW, targetH) => {
                    if (this.resizing.boxLayout.position.x + targetW > this.max-column-count) {
                        targetW = this.max-column-count - this.resizing.boxLayout.position.x
                    } else {
                        if (targetW < 1) {
                            targetW = 1
                        }
                    }
                    if (this.resizing.boxLayout.position.y + targetH > this.max-row-count) {
                        targetH = this.max-row-count - this.resizing.boxLayout.position.y
                    } else {
                        if (targetH < 1) {
                            targetH = 1
                        }
                    }
                    return {
                        targetW,
                        targetH
                    }
                }

                box.$on('resizeStart', evt => {
                    var boxLayout = this.getBoxLayoutById(box.box-id)
                    if (boxLayout.pinned) {
                        return
                    }
                    isResizing = true

                    // find box
                    this.resizing.boxLayout = boxLayout
                    this.placeholder = {
                        ...this.resizing.boxLayout
                    }

                    // clone layout
                    initialLayout = utils.sortLayout(this.layout)

                    this.$emit('resize:start', initialLayout)
                })

                box.$on('resizeUpdate', evt => {
                    if (!isResizing) {
                        return
                    }
                    this.resizing.offset.x = evt.offset.x
                    this.resizing.offset.y = evt.offset.y

                    var resizeBy = this.getPositionByPixel(evt.offset.x, evt.offset.y)
                    if (!utils.isFree(this.pinnedLayout, {
                        ...this.resizing.boxLayout.position,
                        w: this.resizing.boxLayout.position.w + resizeBy.x,
                        h: this.resizing.boxLayout.position.h + resizeBy.y
                    })) {
                        return
                    }

                    let { targetW, targetH } = validateTargetSize(
                        this.resizing.boxLayout.position.w + resizeBy.x,
                        this.resizing.boxLayout.position.h + resizeBy.y
                    )

                    // check if box size has changed
                    if (this.placeholder.position.w === targetW && this.placeholder.position.h === targetH) {
                        return
                    }
                    this.placeholder = utils.updateBoxPosition(this.placeholder, {
                        w: targetW,
                        h: targetH
                    })

                    var newLayout = [ this.placeholder ]
                    initialLayout.forEach((boxLayout) => {
                        if (boxLayout.id === this.resizing.boxLayout.id) {
                            return
                        }
                        newLayout.push(utils.moveBoxToFreePlace(newLayout, boxLayout, this.bubbleUp))
                    })

                    if (this.bubbleUp) {
                        newLayout = utils.layoutBubbleUp(newLayout)
                        this.placeholder = newLayout.find((boxLayout) => {
                            return boxLayout.id === this.resizing.boxLayout.id
                        })
                    }
                    this.updateLayout(newLayout)

                    this.$emit('resize:update', newLayout)
                })

                box.$on('resizeEnd', evt => {
                    if (!isResizing) {
                        return
                    }
                    var resizeBy = this.getPositionByPixel(evt.offset.x, evt.offset.y)
                    if (utils.isFree(this.pinnedLayout, {
                        ...this.resizing.boxLayout.position,
                        w: this.resizing.boxLayout.position.w + resizeBy.x,
                        h: this.resizing.boxLayout.position.h + resizeBy.y
                    })) {
                        let { targetW, targetH } = validateTargetSize(
                            this.resizing.boxLayout.position.w + resizeBy.x,
                            this.resizing.boxLayout.position.h + resizeBy.y
                        )

                        this.placeholder = utils.updateBoxPosition(this.placeholder, {
                            w: targetW,
                            h: targetH
                        })
                    }

                    this.resizing.boxLayout = utils.updateBoxPosition(this.resizing.boxLayout, {
                        w: this.placeholder.position.w,
                        h: this.placeholder.position.h
                    })

                    var newLayout = [ this.resizing.boxLayout ]
                    initialLayout.forEach((boxPosition) => {
                        if (boxPosition.id === this.resizing.boxLayout.id) {
                            return
                        }
                        newLayout.push(utils.moveBoxToFreePlace(newLayout, boxPosition, this.bubbleUp))
                    })

                    if (this.bubbleUp) {
                        newLayout = utils.layoutBubbleUp(newLayout)
                    }
                    this.updateLayout(newLayout)

                    this.resizing.boxLayout = null
                    this.resizing.offset.x = 0
                    this.resizing.offset.y = 0

                    this.placeholder.hidden = true

                    this.$emit('resize:end', newLayout)
                })
            },
            createBoxLayout (...box-ids) {
                box-ids = box-ids.filter(box-id => !this.getBoxLayoutById(box-id))

                if (box-ids.length) {
                    let newLayout = [
                        ...this.layout
                    ]
                    box-ids.forEach(box-id => {
                        newLayout.push(utils.moveBoxToFreePlace(newLayout, {
                            id: box-id,
                            hidden: false,
                            position: {
                                x: 0,
                                y: 0,
                                ...this.default-size
                            }
                        }, this.bubbleUp))
                    })
                    this.updateLayout(newLayout)
                }
            }
        },
        created () {
            List.add(this)
        },
        mounted () {
            this.isMounted = true
            let box-ids = this.$children.map(box => box.$props.box-id)
            this.createBoxLayout(...box-ids)
        },
        beforeDestroy () {
            List.delete(this)
        }
    }
</script>
