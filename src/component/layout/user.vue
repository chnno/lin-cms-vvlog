<template>
  <div class="user">
    <el-dropdown @command="handleCommand">
      <span class="el-dropdown-link">
        <div class="nav-avatar">
          <img :src="user.avatar || defaultAvatar" alt="头像" />
        </div>
      </span>
      <el-dropdown-menu slot="dropdown" class="user-box">
        <div class="user-info">
          <div class="avatar" title="点击修改头像">
            <img :src="user.avatar || defaultAvatar" alt="头像" />
            <label class="mask">
              <i class="iconfont icon-icon-test" style="font-size: 20px;"></i>
              <input ref="avatarInput" type="file" accept="image/*" @change="fileChange" />
            </label>
          </div>
          <div class="text">
            <div class="username" @click="changeNickname" v-if="!nicknameChanged">{{ nickname }}</div>
            <el-input
              placeholder="请输入内容"
              size="small"
              v-else
              v-model="nickname"
              ref="input"
              @blur="blur"
            ></el-input>
          </div>
          <img src="../../assets/image/user/corner.png" class="corner" />
          <div class="info">
            <div class="username">{{ username }}</div>
            <!-- <div class="mid">|</div> -->
            <!-- <div class="desc">{{ groupName }}</div> -->
          </div>
        </div>
        <el-dropdown-item command="home" icon="el-icon-user">我的主页</el-dropdown-item>
        <el-dropdown-item command="main" icon="el-icon-menu">创作者中心</el-dropdown-item>
        <el-dropdown-item command="settings" icon="el-icon-s-tools">设置</el-dropdown-item>
        <el-dropdown-item command="outLogin" icon="el-icon-warning">退出账户</el-dropdown-item>
      </el-dropdown-menu>
    </el-dropdown>
    <!-- 修改头像 -->
    <el-dialog
      title="裁剪"
      :visible.sync="cropVisible"
      width="300px"
      :append-to-body="true"
      :close-on-click-modal="false"
      custom-class="croppa-dialog"
      center
    >
      <div style="text-align: center;">
        <div class="avatar-croppa-container">
          <croppa
            ref="croppa"
            :width="cropRule.width"
            :height="cropRule.height"
            :placeholder="'loading'"
            :zoom-speed="30"
            :disable-drag-and-drop="false"
            :show-remove-button="false"
            :prevent-white-space="true"
            :disable-click-to-choose="false"
            :disable-scroll-to-zoom="false"
            :show-loading="true"
            :quality="quality"
            :initial-image="cropImg"
          ></croppa>
        </div>
        <div style="margin-top: 1em;">通过鼠标滚轮调节头像大小</div>
      </div>
      <div slot="footer" class="dialog-footer">
        <el-button @click="cropVisible = false" size="small">取 消</el-button>
        <el-button type="primary" @click="handleCrop" size="small">确 定</el-button>
      </div>
    </el-dialog>
  </div>
</template>

<script>
import { mapActions, mapGetters } from 'vuex';
import Vue from 'vue';
import Croppa from 'vue-croppa';
import User from '@/lin/model/user';
import 'vue-croppa/dist/vue-croppa.css';
import defaultAvatar from '@/assets/image/user/user.png';

Vue.use(Croppa);

const width = 150;
const height = 150;

export default {
  name: 'user',
  components: {},
  data() {
    return {
      username: null,
      nicknameChanged: false,
      nickname: null,
      groupName: null,
      cropRule: {
        width,
        height,
      },
      imgRule: {
        minWidth: width,
        minHeight: height,
      },
      cropVisible: false,
      cropImg: '',
      croppa: {},
      imgInfo: null,
      quality: 1,
      defaultAvatar,
    };
  },
  computed: {
    ...mapGetters(['user']),
  },
  watch: {
    cropVisible(val) {
      if (!val) {
        this.$refs.croppa.remove();
        this.cropImg = '';
        this.imgInfo = null;
      }
    },
  },
  created() {
    this.init();
  },
  methods: {
    ...mapActions(['loginOut', 'setUserAndState']),
    fileChange(evt) {
      if (evt.target.files.length !== 1) {
        return;
      }
      const imgFile = evt.target.files[0];
      // 验证文件大小是否符合要求, 不大于 5M
      if (imgFile.size > 1024 * 1024 * 5) {
        this.$message.error('文件过大超过5M');
        // 清空输入框
        this.clearFileInput(this.$refs.avatarInput);
        return;
      }

      // 验证图像是否符合要求
      const imgSrc = window.URL.createObjectURL(imgFile);
      const image = new Image();
      image.src = imgSrc;
      image.onload = () => {
        const w = image.width;
        const h = image.height;
        if (w < 50) {
          this.$message.error('图像宽度过小, 请选择大于50px的图像');
          // 清空输入框
          this.clearFileInput(this.$refs.avatarInput);
          return;
        }
        if (h < 50) {
          this.$message.error('图像高度过小, 请选择大于50px的图像');
          // 清空输入框
          this.clearFileInput(this.$refs.avatarInput);
          return;
        }
        // 验证通过, 打开裁剪框
        this.cropImg = imgSrc;
        this.cropVisible = true;
        if (this.$refs.croppa) {
          this.$refs.croppa.refresh();
        }
      };
      image.onerror = () => {
        this.$message.error('获取本地图片出现错误, 请重试');
        // 清空输入框
        this.clearFileInput(this.$refs.avatarInput);
      };
    },
    async handleCrop() {
      // 获取裁剪数据
      const blob = await this.$refs.croppa.promisedBlob('image/jpeg', 0.8);
      // 构造为文件对象
      const file = new File([blob], 'avatar.jpg', {
        type: 'image/jpeg',
      });

      return this.$axios({
        method: 'post',
        url: '/cms/file',
        data: {
          file,
        },
      }).then((res) => {
        // 清空输入框
        this.clearFileInput(this.$refs.avatarInput);
        if (!Array.isArray(res) || res.length !== 1) {
          this.$message.error('头像上传失败, 请重试');
          return false;
        }
        // TODO: 错误码处理
        // if (res.code === 10110) {
        //   throw new Error('文件体积过大')
        // }
        return this.$axios({
          method: 'put',
          url: '/cms/user/avatar',
          data: {
            avatar: res[0].path,
          },
        })
          .then((res) => {
            if (res.code === 0) {
              this.$message({
                type: 'success',
                message: '更新头像成功',
              });
              this.cropVisible = false;
              // 触发重新获取用户信息
              return User.getInformation();
            }
            return Promise.reject(new Error('更新头像失败'));
          })
          .then((res) => {
            this.setUserAndState(res);
          });
      });
    },
    changeNickname() {
      this.nicknameChanged = true;
      setTimeout(() => {
        this.$refs.input.focus();
      }, 200);
    },
    async blur() {
      if (this.nickname) {
        const { user } = this.$store.state;
        if (this.nickname !== user.nickname && this.nickname !== '佚名') {
          this.$axios({
            method: 'put',
            url: '/cms/user/nickname',
            data: {
              nickname: this.nickname,
            },
          })
            .then((res) => {
              if (res.code === 0) {
                this.$message({
                  type: 'success',
                  message: '更新昵称成功',
                });
                // 触发重新获取用户信息
                return User.getInformation();
              }
              this.nickname = user.nickname;
            })
            .then((res) => {
              this.setUserAndState(res);
            });
        }
      }
      this.nicknameChanged = false;
    },
    init() {
      const { user } = this.$store.state;
      this.username = user ? user.username : '未登录';
      // this.groupName = user && user.groupName ? user.groupName : '无角色';
      this.nickname = user && user.nickname ? user.nickname : '佚名';
    },
    async outLogin() {
      await User.logout();
      this.loginOut();
      location.href = '/';
      // window.location.reload(true);
    },

    clearFileInput(ele) {
      ele.value = '';
    },
    handleCommand(command) {
      switch (command) {
        case 'main':
          window.open('#/dashboard');
          break;
        case 'home':
          this.$router.push('/user/' + this.user.id + '/article');
          // window.open("/user/" + this.user.id + "/article");
          break;
        case 'settings':
          this.$router.push('/setting');
          break;
        case 'outLogin':
          this.outLogin();
          break;
      }
    },
  },
};
</script>

<style lang="scss" scoped>
.user {
  height: 40px;

  .el-dropdown-link {
    cursor: pointer;

    .nav-avatar {
      width: 40px;
      height: 40px;
      border-radius: 50%;
      overflow: hidden;
      margin-right: 10px;
      img {
        width: 100%;
      }
    }
  }
}

.user-box {
  width: 326px;
  background-color: none;
  // background: transparent;
  margin-bottom: 0;
  padding-bottom: 0;
  border: none;

  .user-info {
    background-image: url('../../assets/image/user/user-bg.png');
    background-size: 100% 100%;
    transform: translateY(-10px);
    border-top-left-radius: 4px;
    border-top-right-radius: 4px;
    display: flex;
    flex-direction: row;
    padding: 35px 20px 25px 30px;
    z-index: 100;
    position: relative;

    .corner {
      position: absolute;
      right: 18px;
      top: -9px;
      width: 27px;
      height: 10px;
    }

    .avatar {
      width: 80px;
      height: 80px;
      border-radius: 50%;
      cursor: pointer;
      overflow: hidden;
      position: relative;
      img {
        width: 100%;
      }
      .mask {
        opacity: 0;
        transition: all 0.2s;
        position: absolute;
        top: 0;
        left: 0;
        width: 100%;
        height: 100%;
        background: rgba(0, 0, 0, 0.3);
        display: flex;
        justify-content: center;
        align-items: center;
        cursor: pointer;
        color: white;

        input {
          display: none;
        }
      }

      &:hover {
        .mask {
          opacity: 1;
        }
      }
    }

    .text {
      margin-left: 20px;
      color: #fff;
      display: flex;
      flex-direction: column;
      justify-content: center;

      .username {
        margin-bottom: 10px;
        font-size: 16px;
        cursor: pointer;
      }

      .desc {
        font-size: 14px;
        color: rgba(222, 226, 230, 1);
      }
    }

    .info {
      position: absolute;
      bottom: 10px;
      right: 10px;
      display: flex;
      color: #fff;
      font-size: 14px;
      height: 20px;
      line-height: 20px;

      .mid {
        padding: 0 5px;
      }
    }
  }

  .dropdown-box {
    display: flex;
    flex-direction: column;
    justify-content: space-around;
    padding-left: 35px;
    height: 122px;
    color: #596c8e;
    font-size: 14px;
    background: white;
    margin-top: -10px;

    li {
      cursor: pointer;

      &:nth-child(1) {
        margin-top: 20px;
      }

      &:nth-child(2) {
        margin-bottom: 20px;
      }

      i {
        margin-right: 10px;
      }

      &:hover {
        color: $theme !important;

        i {
          color: $theme !important;
        }
      }
    }
  }
}

.popper__arrow {
  display: none !important;
}

.avatar-croppa-container {
  display: inline-block;
  border-color: #3862bc;
  border-style: dashed;
  font-size: 0;
  border-width: 2px;
}
</style>
