<template>
  <div id="tags-view-container" class="tabs-bar-container">
    <el-tabs
      v-model="tabActive"
      type="card"
      class="tabs-content"
      @tab-click="handleTabClick"
      @tab-remove="handleTabRemove"
    >
      <el-tab-pane
        v-for="item in visitedViews"
        :key="item.path"
        :label="item.meta.title"
        :name="item.path"
        :closable="!isAffix(item)"
      ></el-tab-pane>
    </el-tabs>
    <el-dropdown @command="handleCommand">
      <span style="cursor: pointer">
        更多操作
        <i class="el-icon-arrow-down el-icon--right"></i>
      </span>
      <el-dropdown-menu slot="dropdown" class="tabs-more">
        <!-- <el-dropdown-item command="refreshRoute">
          <vab-icon :icon="['fas', 'circle-notch']" />
          刷新
        </el-dropdown-item> -->
        <el-dropdown-item command="closeOtherstabs">
          <i class="el-icon-circle-close"></i>
          关闭其他
        </el-dropdown-item>
        <el-dropdown-item command="closeLefttabs">
          <i class="el-icon-circle-close"></i>
          关闭左侧
        </el-dropdown-item>
        <el-dropdown-item command="closeRighttabs">
          <i class="el-icon-circle-close"></i>
          关闭右侧
        </el-dropdown-item>
        <el-dropdown-item command="closeAlltabs">
          <i class="el-icon-circle-close"></i>
          关闭全部
        </el-dropdown-item>
      </el-dropdown-menu>
    </el-dropdown>
  </div>
</template>

<script>
import path from 'path'

export default {
  data() {
    return {
      visible: false,
      top: 0,
      left: 0,
      selectedTag: {},
      affixTags: [],
      tabActive: "",
    }
  },
  computed: {
    visitedViews() {
      return this.$store.state.tagsView.visitedViews
    },
    routes() {
      return this.$store.state.permission.routes
    },
    theme() {
      return this.$store.state.settings.theme;
    }
  },
  watch: {
    $route:{
      handler(route) {
        this.initTags();
        this.addTags();
        let tabActive = "";
        this.visitedViews.forEach((item, index) => {
          if (item.path === this.$route.path) {
            tabActive = item.path;
          }
        });
        this.tabActive = tabActive;
      },
      immediate: true,
    },
  },
  methods: {
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
    refreshSelectedTag(view) {
      this.$store.dispatch('tagsView/delCachedView', view).then(() => {
        const { fullPath } = view
        this.$nextTick(() => {
          this.$router.replace({
            path: '/redirect' + fullPath
          })
        })
      })
    },
    closeSelectedTag(view) {
      this.$store.dispatch('tagsView/delView', view).then(({ visitedViews }) => {
        if (this.isActive(view)) {
          this.toLastView(visitedViews, view)
        }
      })
    },
    async closeLefttabs() {
      const view = await this.toThisTag();
      await this.$store.dispatch("tagsView/delLeftRoutes", view);
    },
    async closeRighttabs() {
      const view = await this.toThisTag();
      await this.$store.dispatch("tagsView/delRightRoutes", view);
    },
    async closeOthersTags() {
      const view = await this.toThisTag();
      await this.$store.dispatch('tagsView/delOthersViews', view);
    },
    closeAllTags(view) {
      this.$store.dispatch('tagsView/delAllViews').then(({ visitedViews }) => {
        if (this.affixTags.some(tag => tag.path === this.$route.path)) {
          return
        }
        this.toLastView(visitedViews, view)
      })
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
    handleTabClick(tab) {
      const route = this.visitedViews.filter((item, index) => {
        if (tab.index == index) return item;
      })[0];
      if (this.$route.path !== route.path) {
        this.$router.push({
          path: route.path,
          query: route.query,
          fullPath: route.fullPath,
        });
      } else {
        return false;
      }
    },
    async handleTabRemove(tabActive) {
      let view;
      this.visitedViews.forEach((item, index) => {
        if (tabActive == item.path) {
          view = item;
        }
      });
      const { visitedViews } = await this.$store.dispatch(
        "tagsView/delView",
        view
      );
      if (this.isActive(view)) {
        this.toLastView(visitedViews, view);
      }
    },
    async toThisTag() {
      const view = this.visitedViews.filter((item, index) => {
        if (item.path === this.$route.fullPath) {
          return item;
        }
      })[0];
      if (this.$route.path !== view.path) this.$router.push(view);
      return view;
    },
    handleCommand(command) {
      switch (command) {
        case "refreshRoute":
          this.refreshSelectedTag();
          break;
        case "closeOtherstabs":
          this.closeOthersTags();
          break;
        case "closeLefttabs":
          this.closeLefttabs();
          break;
        case "closeRighttabs":
          this.closeRighttabs();
          break;
        case "closeAlltabs":
          this.closeAllTags();
          break;
      }
    },
  }
}
</script>
<style lang="scss" scoped>
@import "~@/assets/styles/variables.scss";
.tabs-bar-container {
  position: relative;
  box-sizing: border-box;
  display: flex;
  align-content: center;
  align-items: center;
  justify-content: space-between;
  height: $base-tabs-bar-height;
  padding-right: $base-padding;
  padding-left: $base-padding;
  user-select: none;
  background: $base-color-white;
  border-top: 1px solid #f6f6f6;

  ::v-deep {
    .fold-unfold {
      margin-right: $base-padding;
    }
  }

  .tabs-content {
    width: calc(100% - 90px);
    height: $base-tag-item-height;

    ::v-deep {
      .el-tabs__nav-next,
      .el-tabs__nav-prev {
        height: $base-tag-item-height;
        line-height: $base-tag-item-height;
      }

      .el-tabs__header {
        border-bottom: 0;

        .el-tabs__nav {
          border: 0;
        }

        .el-tabs__item {
          box-sizing: border-box;
          height: $base-tag-item-height;
          margin-right: 5px;
          line-height: $base-tag-item-height;
          border: 1px solid $base-border-color;
          border-radius: $base-border-radius;
          transition: padding 0.3s cubic-bezier(0.645, 0.045, 0.355, 1) !important;
          &.is-active {
            border: 1px solid $base-color-blue;
            background: $base-color-blue;
            color: $base-color-white;
          }
        }
      }
    }
  }

  .more {
    display: flex;
    align-content: center;
    align-items: center;
    cursor: pointer;
  }
}
</style>
