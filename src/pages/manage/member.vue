<template>
  <qui-page :data-qui-theme="theme" class="member-box">
    <!-- 搜索成员 -->
    <view class="member-box-search">
      <view class="search">
        <view class="search-box">
          <view class="search-box__content">
            <view class="icon-content-search">
              <qui-icon name="icon-search" size="30" color="#bbb"></qui-icon>
            </view>
            <input
              type="text"
              class="search-box__content-input"
              placeholder-class="input-placeholder"
              :placeholder="i18n.t('manage.searchMembers')"
              @input="searchInput"
              :value="searchText"
            />
            <view @tap="cancelSearch" v-if="searchText" class="search-box__content-delete">
              <qui-icon name="icon-close1" size="32" color="#ccc"></qui-icon>
            </view>
          </view>
          <view class="search-box__cancel" v-if="searchText" @tap="cancelSearch">
            <text>{{ i18n.t('search.cancel') }}</text>
          </view>
        </view>
      </view>
    </view>
    <!-- 成员列表 -->
    <view class="member-box__list">
      <!-- #ifdef MP-WEIXIN -->
      <scroll-view
        class="wx-scroll"
        scroll-y="true"
        scroll-with-animation="true"
        @scrolltolower="pullDown"
      >
        <checkbox-group @change="changeCheck">
          <label v-for="item in userListShow" :key="item.id">
            <qui-avatar-cell
              center
              right-color="#aaa"
              :mark="item.id"
              :title="item.username"
              :label="handleGroups(item)"
              :value="item.status == 1 ? i18n.t('manage.disable') : i18n.t('manage.normal')"
              :icon="item.avatarUrl"
              :is-real="item.isReal"
            >
              <checkbox
                slot="rightIcon"
                :value="JSON.stringify(item)"
                :disabled="item.id === currentLoginId ? true : false"
                :checked="checkAvatar.findIndex(value => value.id === item.id) > -1"
              ></checkbox>
            </qui-avatar-cell>
          </label>
        </checkbox-group>
        <qui-load-more :status="loadingTypeShow"></qui-load-more>
      </scroll-view>
      <!-- #endif -->
      <!-- #ifdef H5-->
      <scroll-view
        class="h5-scroll"
        scroll-y="true"
        scroll-with-animation="true"
        @scrolltolower="pullDown"
      >
        <checkbox-group @change="changeCheck">
          <label v-for="item in userListShow" :key="item.id">
            <qui-avatar-cell
              center
              right-color="#aaa"
              :mark="item.id"
              :title="item.username"
              :label="handleGroups(item)"
              :value="item.status == 1 ? i18n.t('manage.disable') : i18n.t('manage.normal')"
              :icon="item.avatarUrl"
              :is-real="item.isReal"
            >
              <checkbox
                slot="rightIcon"
                :value="JSON.stringify(item)"
                :disabled="item.id === currentLoginId ? true : false"
                :checked="checkAvatar.findIndex(value => value.id === item.id) > -1"
              ></checkbox>
            </qui-avatar-cell>
          </label>
        </checkbox-group>
        <qui-load-more :status="loadingTypeShow"></qui-load-more>
      </scroll-view>
      <!-- #endif -->
    </view>
    <view class="member-box__ft">
      <qui-button
        size="large"
        :type="Boolean(checkAvatar.length < 1) ? 'default' : 'primary'"
        :disabled="Boolean(checkAvatar.length < 1)"
        @click="getCheckMember"
      >
        {{
          checkAvatar.length &lt; 1
            ? i18n.t('manage.notSelected')
            : i18n.t('manage.selected') + '（' + checkAvatar.length + '）'
        }}
      </qui-button>
    </view>
    <!-- 成员管理弹窗 -->
    <uni-popup ref="popup" type="bottom">
      <scroll-view scroll-y style="max-height: 968rpx;">
        <view class="popup-wrap">
          <view class="popup-wrap-con">
            <view @click="modifyGroupName(item)" v-for="item in groupList" :key="item._jv.id">
              <view class="popup-wrap-con-text">{{ item.name }}</view>
              <view class="popup-wrap-con-line"></view>
            </view>
            <view
              @click="modifyGroupName({}, 1)"
              v-if="forums.other && forums.other.can_edit_user_status"
            >
              <view class="popup-wrap-con-text">
                {{ i18n.t('manage.disable') }}
              </view>
              <view class="popup-wrap-con-line"></view>
            </view>
            <view
              @click="modifyGroupName({}, 2)"
              v-if="forums.other && forums.other.can_edit_user_status"
            >
              <view class="popup-wrap-con-text">
                {{ i18n.t('manage.clearDisable') }}
              </view>
              <view class="popup-wrap-con-line"></view>
            </view>
          </view>
          <view class="popup-wrap-space"></view>
          <text class="popup-wrap-btn" @click="cancel">{{ i18n.t('home.cancel') }}</text>
        </view>
      </scroll-view>
    </uni-popup>
  </qui-page>
</template>

<script>
import { debounce } from 'lodash';
import forums from '@/mixin/forums';

export default {
  mixins: [forums],
  data() {
    return {
      searchText: '', // 输入的用户名
      loadingType: 'more', // 上拉加载状态
      searchLoadingType: 'more', // 搜索上拉加载状态
      pageSize: 20, // 每页20条数据
      pageNum: 1, // 当前页数
      searchPageNum: 1, // 搜索的当前页数
      userList: [], // 用户数据
      searchUserList: [], // 搜索的用户数据
      isSearch: false, // 是否搜索
      groupList: [], // 用户角色
      checkAvatar: [], // 选择的用户
    };
  },
  onLoad() {
    this.searchUser();
    this.getGroupList();
    uni.setNavigationBarTitle({
      title: this.i18n.t('manage.manageMembers'),
    });
  },
  computed: {
    // 获取当前登录的id
    currentLoginId() {
      const userId = this.$store.getters['session/get']('userId');
      return parseInt(userId, 10);
    },
    userListShow() {
      return this.isSearch ? this.searchUserList : this.userList;
    },
    loadingTypeShow() {
      return this.isSearch ? this.searchLoadingType : this.loadingType;
    },
  },
  methods: {
    // eslint-disable-next-line
    searchInput: debounce(function(e) {
      if (e && e.target) {
        this.isSearch = true;
        this.searchPageNum = 1;
        this.searchUserList = [];
        this.searchUser(e.target.value);
      }
    }, 800),
    cancelSearch() {
      this.isSearch = false;
      this.searchText = '';
    },
    handleGroups(data) {
      let groups = [];
      if (data.groups && data.groups.length > 0) {
        groups = data.groups.filter(item => item.isDisplay);
      }
      if (groups.length > 0) {
        return groups[0].name;
      }
      return '';
    },
    // 调用 获取所有用户组 接口
    getGroupList() {
      if (this.forums.other && !this.forums.other.can_edit_user_group) {
        this.groupList = [];
        return;
      }
      this.$store.dispatch('jv/get', 'groups').then(res => {
        if (res._jv) {
          delete res._jv;
          this.groupList = res;
        }
      });
    },
    // 调用 搜索 接口
    searchUser(val = '') {
      this.searchText = val;
      const params = {
        include: 'groups',
        'page[number]': this.pageNum,
        'page[limit]': this.pageSize,
        'filter[username]': `*${this.searchText}*`,
      };
      if (this.searchText === '') {
        this.$store.dispatch('jv/get', ['users', { params }]).then(res => {
          if (res && res._jv) {
            delete res._jv;
            if (this.isSearch) {
              this.searchUserList = [...this.searchUserList, ...res];
            } else {
              this.userList = [...this.userList, ...res];
            }
            this.loadingType = res.length === this.pageSize ? 'more' : 'nomore';
          }
        });
      } else {
        params['page[number]'] = this.searchPageNum;
        this.$store.dispatch('jv/get', ['users', { params }]).then(res => {
          if (res && res._jv) {
            delete res._jv;
            this.searchUserList = [...this.searchUserList, ...res];
            this.searchLoadingType = res.length === this.pageSize ? 'more' : 'nomore';
          }
        });
      }
    },
    // 上拉加载
    pullDown() {
      if (this.loadingTypeShow !== 'more') {
        return;
      }
      if (this.isSearch) {
        this.searchPageNum += 1;
      } else {
        this.pageNum += 1;
      }
      this.searchUser(this.searchText);
    },
    // 调用 批量修改用户的用户组 接口
    modifyGroupName(item, status) {
      const data = [];
      if (this.checkAvatar && this.checkAvatar.length > 0) {
        for (let i = 0; i < this.checkAvatar.length; i += 1) {
          if (status) {
            data.push({
              attributes: {
                id: this.checkAvatar[i].id,
                status: status === 1 ? 1 : 0,
              },
            });
          } else {
            data.push({
              attributes: {
                id: this.checkAvatar[i].id,
                groupId: parseInt(item._jv.id, 10),
              },
            });
          }
        }
      }
      const params = [
        {
          _jv: {
            type: 'users',
          },
        },
        {
          data: {
            data,
          },
        },
      ];
      this.$store.dispatch('jv/patch', params).then(res => {
        if (res) {
          this.getGroupList();
          this.pageNum = 1;
          this.getList();
          this.$refs.popup.close();
        }
      });
    },
    getList() {
      this.searchText = '';
      this.isSearch = false;
      const params = {
        include: 'groups',
        'page[number]': this.pageNum,
        'page[limit]': this.pageSize,
        'filter[username]': `*${this.searchText}*`,
      };
      this.$store.dispatch('jv/get', ['users', { params }]).then(res => {
        if (res && res._jv) {
          delete res._jv;
          this.userList = res;
          this.checkAvatar.splice(0, this.checkAvatar.length);
          this.loadingType = res.length === this.pageSize ? 'more' : 'nomore';
        }
      });
    },
    changeCheck(e) {
      this.checkAvatar = [];
      e.detail.value.forEach(item => {
        this.checkAvatar.push(JSON.parse(item));
      });
    },
    getCheckMember() {
      this.$refs.popup.open();
    },
    // 点击取消按钮
    cancel() {
      this.$refs.popup.close();
    },
  },
};
</script>

<style lang="scss" scoped>
@import '@/styles/base/theme/fn.scss';
@import '@/styles/base/variable/global.scss';

.member-box-search {
  .search {
    position: fixed;
    top: 0rpx;
    z-index: 99;
    width: 100%;
    /* #ifdef H5 */
    margin-top: 44px;
    /* #endif */
  }

  .search-box {
    background-color: --color(--qui-BG-2);
  }
}

.member-box {
  width: 100%;

  &__list {
    .h5-scroll {
      position: fixed;
      top: 210rpx;
      right: 0rpx;
      bottom: 160rpx;
      left: 0rpx;
      width: 100%;
      background-color: --color(--qui-BG-2);
      .loading-text {
        height: 100rpx;
        font-size: 28rpx;
        line-height: 100rpx;
        text-align: center;
      }
    }
    .wx-scroll {
      position: fixed;
      top: 120rpx;
      right: 0rpx;
      bottom: 160rpx;
      left: 0rpx;
      width: 100%;
      background-color: --color(--qui-BG-2);
      .loading-text {
        height: 100rpx;
        font-size: 28rpx;
        line-height: 100rpx;
        text-align: center;
      }
    }
  }
  &__ft {
    position: fixed;
    bottom: 0;
    width: 100%;
    padding: 40rpx;
    background-color: --color(--qui-BG-2);
    box-sizing: border-box;
  }
}

.popup-wrap {
  display: flex;
  flex-direction: column;
  background: --color(--qui-BG-2);
  border-radius: 10rpx 10rpx 0rpx 0rpx;

  &-con {
    border-radius: 10rpx 10rpx 0rpx 0rpx;

    &-text {
      width: 100%;
      height: 100rpx;
      font-size: $fg-f5;
      line-height: 100rpx;
      text-align: center;
    }

    &-line {
      border: 2rpx solid --color(--qui-BG-ED);
    }
  }

  &-space {
    border: 8rpx solid --color(--qui-BG-ED);
  }

  &-btn {
    width: 100%;
    height: 100rpx;
    font-size: $fg-f4;
    line-height: 100rpx;
    text-align: center;
  }
}
</style>
