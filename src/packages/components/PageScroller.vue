<template>
    <div class="page-scroller" :class="{'page-scroller-overflow-hidden':overflowHidden}" :id="id || `PageScroller-${scrollerKey}`"
        :data-key="scrollerKey">
        <div class="page-scroller-top-placeholder">
            <div class="page-scroller-top-placeholder-contain">
                {{style}}
                <h1>下拉刷新</h1>
            </div>
        </div>
        <div class="page-scroller-contain" :class="{'page-scroller-transition':touches.length==0}" :style="style">
            <div>{{tips}}</div>
            <slot></slot>
        </div>
    </div>
</template>

<script>
    export default {
        name: "PageScroller",
        data() {
            return {
                scrollerKey: "",
                isAtScrollTop: true, //是否滚动条到达顶部，继续下拉刷新
                style: "",
                overflowHidden: false,
                touches: [],
                tips: "",
            };
        },
        props: {
            // 记录当前容器的唯一标记，用来创建key:value数据记录容器标记
            // 若不传入，会默认获取当前的$route.fullPath
            uniqueKey: {
                type: String,
                default: "",
            },
            id: {
                type: String,
                default: "",
            }
        },
        mounted() {

            let createUniqueKey = () => {
                if (this.$route.meta && ~~this.$route.meta.PageScrollerIndex >= 0) {
                    this.$route.meta.PageScrollerIndex = ~~this.$route.meta.PageScrollerIndex + 1;
                }
                let index = this.$route.meta.PageScrollerIndex;
                return this.$route.fullPath + `-${index}`;
            }

            this.scrollerKey = this.uniqueKey || createUniqueKey();

            // 初始化全局变量
            window.PageScroller = window.PageScroller || {};
            window.PageScroller[this.scrollerKey] = {};

            this.addScrollEvent();

            this.addTouchEvent();

        },
        methods: {
            addTouchEvent() {
                let that = this;

                let touchStartY, touchStartScrollTop;

                this.style = `transform:translate3d(0,0,0)`;

                this.$el.addEventListener("touchstart", function (e) {
                    let touches = e.touches || [],
                        touchOne = touches[0],
                        touchOneX = touchOne.pageX,
                        touchOneY = touchOne.pageY;

                    that.overflowHidden = false;
                    that.touches = touches;

                    touchStartScrollTop = this.scrollTop;
                    touchStartY = touchOneY;
                    // console.log(touchOneY);
                    // e.preventDefault();
                    e.stopPropagation();
                })

                this.$el.addEventListener("touchmove", function (e) {
                    let touches = e.touches || [],
                        touchOne = touches[0],
                        touchOneX = touchOne.pageX,
                        touchOneY = touchOne.pageY,
                        touchOneDist = touchOneY - touchStartY;

                    if (that.isAtScrollTop) {
                        e._isScroller = true;
                    }

                    if (touchOneDist < 0) {
                        e._isScroller = false;
                    }

                    if (touchOneDist > 0 && that.isAtScrollTop) {
                        console.log(4)
                        e._isScroller = true;
                        if (touchOneDist - touchStartScrollTop < 0) {
                            that.style = `transform:translate3d(0,0,0)`;
                        } else {
                            that.style = `transform:translate3d(0,${touchOneDist-touchStartScrollTop}px,0)`;

                        }
                    } else {
                        if (that.overflowHidden) {
                            that.overflowHidden = false;
                            that.style = `transform:translate3d(0,0,0)`;
                        }
                    }
                    // console.warn(touchOneDist, that.isAtScrollTop, that.topPlaceHolderHeight, this
                    //     .scrollTop)
                    // e.stopPropagation();
                    // console.log(11)
                })

                this.$el.addEventListener("touchend", function (e) {
                    if (e.touches && e.touches.length == 0) {
                        // 松开手指参数复位
                        that.style = `transform:translate3d(0,0,0)`;
                        that.overflowHidden = false;
                        that.touches = [];
                    }

                    // console.log(touchOneY);
                    // e.preventDefault();
                    e.stopPropagation();
                })

                document.body.addEventListener('touchmove', function (evt) {
                    //In this case, the default behavior is scrolling the body, which
                    //would result in an overflow.  Since we don't want that, we preventDefault.
                    if (evt._isScroller) {
                        evt.preventDefault()
                    }
                    // that.tips = evt._isScroller;

                })

                // 监听滚动到顶端
                this.$el.addEventListener("scroll", function (e) {

                    // 到达了滚动条顶部
                    if (this.scrollTop <= 1) {
                        that.isAtScrollTop = true;

                        if (that.touches && that.touches.length > 0) {
                            // 触摸状态的手指往下滑
                            that.overflowHidden = true
                        } else {
                            // 松开手指后的滚动条持续滚动
                        }
                    } else {
                        that.isAtScrollTop = false;
                    }
                })
            },
            // 添加滚动监听事件，记录当前el滚动坐标
            addScrollEvent() {

                // 兼容IOS的顶端滚动bug
                setTimeout(() => {
                    this.$el.scrollTop = 1;
                }, 1)

                console.log(this)

                let that = this;
                this.$el.addEventListener("scroll", function (e) {
                    // console.log(this.scrollTop)
                    let PageScroller = window.PageScroller;
                    let scrollerKey = that.scrollerKey;
                    let scrollerObj = PageScroller[scrollerKey] || {};
                    scrollerObj["x"] = this.scrollLeft;
                    scrollerObj["y"] = this.scrollTop;
                    scrollerObj["el"] = this;
                    window.PageScroller[scrollerKey] = scrollerObj;

                    // 到达顶部，兼容IOS的顶端滚动bug
                    if (this.scrollTop <= 0) {
                        console.log("到达顶部");
                        this.scrollTop = 1;
                    }

                    // 到达底部，兼容IOS的顶端滚动bug
                    if (this.scrollHeight <= this.clientHeight + this.scrollTop) {
                        console.log("到达底部");
                        this.scrollTop = (this.scrollHeight - this.clientHeight - 1);
                    }

                })
            },
            // 根据key值进行定位
            resetScroll(scrollerKey) {
                let PageScroller = window.PageScroller;
                let scrollerObj = PageScroller[scrollerKey];
                if (scrollerObj) {
                    let el = scrollerObj["el"] || this.$el;
                    el.scrollLeft = scrollerObj["x"] || 0;
                    el.scrollTop = scrollerObj["y"] || 0;
                }

            }
        },
        watch: {
            '$route'(to, from) {
                this.resetScroll(this.scrollerKey);
            }
        },
        inheritAttrs: true
    };
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped lang="scss">
    .page-scroller {

        &.page-scroller-overflow-hidden {
            overflow: hidden;
        }

        .page-scroller-top-placeholder {
            width: 100%;
            top: 0;
            left: 0;
            // overflow: hidden;
            position: relative;

            .page-scroller-top-placeholder-contain {
                position: absolute;
                width: 100%;
                top: 0;
            }
        }

        .page-scroller-contain {
            position: relative;
            background: inherit;
            // will-change: transform;

            &.page-scroller-transition {
                transition: all .3s;
            }
        }
    }
</style>