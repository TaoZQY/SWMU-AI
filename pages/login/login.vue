<template>
  <view class="login-container">
    <view class="form">
      <!-- 添加 Logo 图片 -->
      <view class="logo-container">
        <image class="logo" src="/static/swmu-logo.jpg" mode="widthFix"></image>
      </view>
      <input class="input" v-model="username" placeholder="请输入用户名" />
      <input class="input" v-model="password" type="password" placeholder="请输入密码" />
      <button class="login-button" @click="handleLogin">登录</button>
    </view>
  </view>
</template>

<script>
  export default {
    data() {
      return {
        username: '',
        password: '',
      };
    },
    methods: {
      handleLogin() {
        if (!this.username) {
          uni.showToast({
            title: '用户名不能为空',
            icon: 'none',
          });
          return;
        }
        if (!this.password) {
          uni.showToast({
            title: '密码不能为空',
            icon: 'none',
          });
          return;
        }

        // 模拟登录验证
        if (this.username === 'swmulqy' && this.password === 'swmulqy821') {
          uni.showLoading({
            title: '登录中...',
          });

          setTimeout(() => {
            uni.hideLoading();
            uni.showToast({
              title: '登录成功',
              icon: 'success',
            });
            // 登录成功后进行页面跳转
            uni.navigateTo({
              url: '/pages/index/index' // 跳转到首页
            });
          }, 500);
        } else {
          uni.showToast({
            title: '用户名或密码错误',
            icon: 'none',
          });
        }
      },
    },
  };
</script>

<style scoped>
  .login-container {
     display: flex;
      justify-content: center;
      align-items: center;
      height: 100vh;
      background-color: #f5f5f5;
  }

  .logo-container {
    display: flex;
    justify-content: center;
    margin-bottom: 20px;
    /* Logo与表单输入框之间的间距 */
  }

  .logo {
    width: 100px;
    /* 设置Logo的宽度 */
  }

  .form {
    width: 80%;
    max-width: 400px;
    background-color: #ffffff;
    padding: 20px;
    border-radius: 10px;
    box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
  }

  .input {
    width: 100%;
    margin-bottom: 15px;
    padding: 10px;
    border: 1px solid #ddd;
    border-radius: 5px;
    font-size: 16px;
  }

  .login-button {
    width: 100%;
    padding: 10px;
    background-color: #4f75ff;
    color: #ffffff;
    border: none;
    border-radius: 5px;
    font-size: 16px;
  }
</style>
