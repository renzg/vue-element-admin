<template>
  <div id="tags-view-container" class="tags-view-container">
    <vue-tabs-chrome
      ref="tab"
      v-model="tab"
      :tabs="tabs"
      :on-close="onClose"
      @click="handleTabClick"
      @dragend="handleDragEnd"
      @contextmenu.prevent.native="handleRightClick"
    />
    <!-- <router-link
      v-for="tag in visitedViews"
      ref="tag"
      :key="tag.path"
      :class="isActive(tag)?'active':''"
      :to="{ path: tag.path, query: tag.query, fullPath: tag.fullPath }"
      tag="div"
      class="chrome-tab"
      @click.middle.native="!isAffix(tag)?closeSelectedTag(tag):''"
      @contextmenu.prevent.native="openMenu(tag,$event)"
    >
      <div class="chrome-tab-background">
        <svg version="1.1" xmlns="http://www.w3.org/2000/svg"><defs><symbol id="chrome-tab-geometry-left" viewBox="0 0 214 36"><path d="M17 0h197v36H0v-2c4.5 0 9-3.5 9-8V8c0-4.5 3.5-8 8-8z"/></symbol><symbol id="chrome-tab-geometry-right" viewBox="0 0 214 36"><use xlink:href="#chrome-tab-geometry-left"/></symbol><clipPath id="crop"><rect class="mask" width="100%" height="100%" x="0"/></clipPath></defs><svg width="52%" height="100%"><use xlink:href="#chrome-tab-geometry-left" width="214" height="36" class="chrome-tab-geometry"/></svg><g transform="scale(-1, 1)"><svg width="52%" height="100%" x="-100%" y="0"><use xlink:href="#chrome-tab-geometry-right" width="214" height="36" class="chrome-tab-geometry"/></svg></g></svg>
      </div>
      <div class="chrome-tab-content">
        <div class="chrome-tab-title">{{ tag.title }}</div>
        <div class="chrome-tab-drag-handle"></div>
        <div class="chrome-tab-close"></div>
      </div>
      <span v-if="!isAffix(tag)" class="el-icon-close" @click.prevent.stop="closeSelectedTag(tag)" />
    </router-link> -->
    <ul v-show="visible" :style="{left:left+'px',top:top+'px'}" class="contextmenu">
      <!-- <li @click="refreshSelectedTag(selectedTag)">Refresh</li> -->
      <li v-if="!isAffix(selectedTag)" @click="closeSelectedTag(selectedTag)">Close</li>
      <li @click="closeOthersTags">Close Others</li>
      <li @click="closeAllTags(selectedTag)">Close All</li>
    </ul>
  </div>
</template>

<script>
import path from 'path'
import { mapGetters } from 'vuex'
export default {
  components: { },
  data() {
    return {
      visible: false,
      top: 0,
      left: 0,
      selectedTag: {},
      affixTags: [],
      tab: '',
      tabs: []
    }
  },
  computed: {
    ...mapGetters([
      'visitedViews'
    ]),
    routes() {
      return this.$store.state.permission.routes
    }
  },
  watch: {
    $route(value) {
      if (!value.name) return
      var result = this.visitedViews.some(function(item) {
        if (item.name === value.name) {
          return true
        }
      })
      if (!result) {
        var newTabs = [
          {
            label: value.name,
            key: value.name,
            closable: !this.isAffix(value)
          }
        ]
        this.$refs.tab.addTab(...newTabs)
      }
      this.tab = value.name
      this.addTags()
    },
    visible(value) {
      if (value) {
        document.body.addEventListener('click', this.closeMenu)
      } else {
        document.body.removeEventListener('click', this.closeMenu)
      }
    }
  },
  mounted() {
    // this.layoutTabs()
    this.$nextTick(() => {
      if (this.visitedViews.length > 0) {
        this.visitedViews.forEach((v, i) => {
          var newTabs = [
            {
              label: v.name,
              key: v.name,
              closable: !this.isAffix(v)
            }
          ]
          this.$refs.tab.addTab(...newTabs)
        })
        this.tab = this.tabs[0].key
      }
    })
    this.initTags()
    this.addTags()
  },
  methods: {
    onClose(tab, key, index) {
      // do not update tab key
      this.visitedViews.forEach((v, i) => {
        if (v.name === key) {
          this.$store.dispatch('tagsView/delView', v)
        }
      })
      // 判断关闭的是否是当前页
      if (this.tab === key) {
        if (this.tabs[index + 1]) {
          this.visitedViews.forEach((v1, i1) => {
            if (v1.name === this.tabs[index + 1].key) {
              this.$router.push({ path: v1.path, query: v1.query, fullPath: v1.fullPath })
            }
          })
        } else {
          this.visitedViews.forEach((v2, i2) => {
            if (v2.name === this.tabs[index - 1].key) {
              this.$router.push({ path: v2.path, query: v2.query, fullPath: v2.fullPath })
            }
          })
        }
      }
      return true
    },
    handleTabClick(e, tab, index) {
      this.visitedViews.forEach((v, i) => {
        if (v.name === tab.key) {
          this.$router.push({ path: v.path, query: v.query, fullPath: v.fullPath })
        }
      })
    },
    handleDragEnd(e, tab) {
      this.handleTabClick(e, tab)
    },
    isActive(route) {
      return route.path === this.$route.path
    },
    isAffix(tag) {
      return tag.meta && tag.meta.affix
    },
    filterAffixTags(routes, basePath = '/') {
      let tags = []
      routes.forEach(route => {
        if (route.meta && route.meta.affix) {
          const tagPath = path.resolve(basePath, route.path)
          tags.push({
            fullPath: tagPath,
            path: tagPath,
            name: route.name,
            meta: { ...route.meta }
          })
        }
        if (route.children) {
          const tempTags = this.filterAffixTags(route.children, route.path)
          if (tempTags.length >= 1) {
            tags = [...tags, ...tempTags]
          }
        }
      })
      return tags
    },
    initTags() {
      const affixTags = this.affixTags = this.filterAffixTags(this.routes)
      for (const tag of affixTags) {
        // Must have tag name
        if (tag.name) {
          this.$store.dispatch('tagsView/addVisitedView', tag)
        }
      }
    },
    addTags() {
      const { name } = this.$route
      if (name) {
        this.$store.dispatch('tagsView/addView', this.$route)
      }
      return false
    },
    moveToCurrentTag() {
      const tags = this.$refs.tag
      this.$nextTick(() => {
        for (const tag of tags) {
          if (tag.to.path === this.$route.path) {
            this.$refs.scrollPane.moveToTarget(tag)
            // when query is different then update
            if (tag.to.fullPath !== this.$route.fullPath) {
              this.$store.dispatch('tagsView/updateVisitedView', this.$route)
            }
            break
          }
        }
      })
    },
    // refreshSelectedTag(view) {
    //   this.$store.dispatch('tagsView/delCachedView', view).then(() => {
    //     const { fullPath } = view
    //     this.$nextTick(() => {
    //       this.$router.replace(view.path)
    //     })
    //   })
    // },
    closeSelectedTag(view) {
      this.$refs.tab.removeTab(view.name)
      this.$store.dispatch('tagsView/delView', view).then(({ visitedViews }) => {
        if (this.tab === view.name) {
          this.toLastView(visitedViews, view)
        }
      })
    },
    closeOthersTags() {
      this.$router.push(this.selectedTag)
      var delArr = []
      this.tabs.forEach(v => {
        if (v.key !== this.selectedTag.name && v.closable) {
          delArr.push(v.key)
        }
      })
      delArr.forEach(v => {
        this.$refs.tab.removeTab(v)
      })
    },
    closeAllTags(view) {
      var delArr = []
      this.tabs.forEach(v => {
        if (v.closable) {
          delArr.push(v.key)
        }
      })
      delArr.forEach(v => {
        this.$refs.tab.removeTab(v)
      })
      this.$router.push(this.visitedViews.slice(-1)[0].fullPath)
    },
    toLastView(visitedViews, view) {
      const latestView = visitedViews.slice(-1)[0]
      if (latestView) {
        this.$router.push(latestView.fullPath)
      } else {
        // now the default is to redirect to the home page if there is no tags-view,
        // you can adjust it according to your needs.
        if (view.name === 'Dashboard') {
          // to reload home page
          this.$router.replace({ path: '/redirect' + view.fullPath })
        } else {
          this.$router.push('/')
        }
      }
    },
    handleRightClick(e) {
      const menuMinWidth = 105
      const offsetLeft = this.$el.getBoundingClientRect().left // container margin left
      const offsetWidth = this.$el.offsetWidth // container width
      const maxLeft = offsetWidth - menuMinWidth // left boundary
      const left = e.clientX - offsetLeft + 15 // 15: margin right

      if (left > maxLeft) {
        this.left = maxLeft
      } else {
        this.left = left
      }

      this.top = e.clientY
      this.visible = true
      this.visitedViews.forEach((item, index) => {
        if (item.name === e.target.innerText) {
          this.selectedTag = item
        }
      })
    },
    closeMenu() {
      this.visible = false
    },
    handleScroll() {
      this.closeMenu()
    }
  }
}
</script>

<style lang="scss" scoped>
.tags-view-container {
  height: 45px;
  width: 100%;
  background: #dee1e6;
  border-bottom: 1px solid #d8dce5;
  box-shadow: 0 1px 3px 0 rgba(0, 0, 0, .12), 0 0 3px 0 rgba(0, 0, 0, .04);
  .tags-view-wrapper {
    .chrome-tab {
      display: inline-block;
      position: relative;
      cursor: pointer;
      height: 26px;
      width: 180px;
      line-height: 26px;
      border: 1px solid #d8dce5;
      color: #495060;
      background: #fff;
      padding: 0 8px;
      font-size: 12px;
      margin-left: 5px;
      margin-top: 4px;
    }
  }
  .chrome-tab-background {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    overflow: hidden;
    pointer-events: none;
  }
  .chrome-tab-background > svg {
    width: 100%;
    height: 100%;
  }
  .chrome-tab-background > svg .chrome-tab-geometry {
    fill: #f4f5f6;
  }
  .chrome-tab[active] {
    z-index: 5;
  }
  .chrome-tab[active] .chrome-tab-background > svg .chrome-tab-geometry {
    fill: #fff;
  }
  .chrome-tab:not([active]) .chrome-tab-background {
    transition: opacity 0.2s ease;
    opacity: 0;
  }
  .contextmenu {
    margin: 0;
    background: #fff;
    z-index: 3000;
    position: absolute;
    list-style-type: none;
    padding: 5px 0;
    border-radius: 4px;
    font-size: 12px;
    font-weight: 400;
    color: #333;
    box-shadow: 2px 2px 3px 0 rgba(0, 0, 0, .3);
    li {
      margin: 0;
      padding: 7px 16px;
      cursor: pointer;
      &:hover {
        background: #eee;
      }
    }
  }
}
</style>

<style lang="scss">
.tabs-label {
  font-size: 12px;
  cursor: pointer;
}
//reset element css of el-icon-close
.tags-view-wrapper {
  .tags-view-item {
    .el-icon-close {
      width: 16px;
      height: 16px;
      vertical-align: 2px;
      border-radius: 50%;
      text-align: center;
      transition: all .3s cubic-bezier(.645, .045, .355, 1);
      transform-origin: 100% 50%;
      &:before {
        transform: scale(.6);
        display: inline-block;
        vertical-align: -3px;
      }
      &:hover {
        background-color: #b4bccc;
        color: #fff;
      }
    }
  }
}
</style>
