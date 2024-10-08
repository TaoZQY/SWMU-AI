<template>
  <view class="container">
    <!-- 选择溶液类型的下拉框 -->
    <picker mode="selector" :range="solutionTypes" @change="onSolutionTypeChange">
      <view class="picker">
        CHOOSE SOLVENTS: {{ selectedSolutionType }}
      </view>
    </picker>
    <!-- 显示图片和RGB信息 -->
    <view class="images-container">
      <view v-for="(image, index) in images" :key="index" class="image-wrapper">
        <!-- 隐藏的canvas，用于获取RGB数据 -->
        <canvas :canvas-id="'canvas' + index"
          :style="{ width: image.canvasWidth + 'px', height: image.canvasHeight + 'px' }"
          @touchstart="startDrawing($event, index)" @touchmove="drawRectangle($event, index)"
          @touchend="endDrawing(index)">
        </canvas>
        <!-- 操作按钮 -->
        <view class="action-buttons">
          <button class="action-button delete-button" @click="deleteImage(index)">×</button>
          <button class="action-button save-button" @click="saveImageInfo(image)">&#x1F4BE;</button>
        </view>
        <!-- 显示RGB信息 -->
        <view v-if="image.rgbInfo" class="rgb-info">
          <!-- RGB 信息上下排列并居中 -->
          <view class="rgb-values">
            <text style="color: red">R: {{ image.rgbInfo.r }}</text>
            <text style="color: green">G: {{ image.rgbInfo.g }}</text>
            <text style="color: blue">B: {{ image.rgbInfo.b }}</text>
          </view>

          <!-- 水分信息 -->
          <view class="fraction-value">
            <text :style="{ color: concentrationColor }">WATER FRACTION: {{ image.concentration }}</text>
          </view>
        </view>
      </view>
    </view>
    <!-- 上传按钮 -->
    <button class="upload-button" type="primary" @click="uploadImage">DETECT</button>

  </view>
</template>

<script>
  export default {
    onLoad() {
      uni.setLocale('en');
    },
    data() {
      return {
        images: [], // 存储选择的图片路径和RGB信息
        canvasWidth: 200,
        canvasHeight: 200,
        solutionTypes: ['NPA', 'ACN', 'EtOH', 'IPA', 'THF'],
        selectedSolutionType: '',
        concentrationColor: '#FF4500',
        isDrawing: false, // 是否正在绘制矩形框
        startX: 0, // 矩形框起始点X坐标
        startY: 0, // 矩形框起始点Y坐标
        endX: 0, // 矩形框结束点X坐标
        endY: 0 // 矩形框结束点Y坐标
      };
    },
    methods: {
      onSolutionTypeChange(e) {
        this.selectedSolutionType = this.solutionTypes[e.detail.value];
      },
      uploadImage() {
        if (!this.selectedSolutionType) {
          uni.showToast({
            title: 'Please select the solution first',
            icon: 'none'
          });
          return;
        }
        uni.chooseImage({
          count: 4, // 支持多张图片上传
          sizeType: ['original', 'compressed'],
          sourceType: ['album', 'camera'],
          success: (res) => {
            this.images = res.tempFilePaths.map((filePath, index) => ({
              imageUrl: filePath,
              canvasWidth: 200,
              canvasHeight: 200,
              rgbInfo: null,
              canvasId: 'canvas' + index
            }));
            this.$nextTick(() => {
              this.images.forEach((image, index) => {
                uni.getImageInfo({
                  src: image.imageUrl,
                  success: (imgInfo) => {
                    this.drawImageAndCalculateRGB(imgInfo.width, imgInfo.height, index);
                  },
                  fail: (err) => {
                    console.error('获取图片信息失败', err);
                  }
                });
              });
            });
          },
          fail: (err) => {
            console.error('选择图片失败', err);
          }
        });
      },
      drawImageAndCalculateRGB(imgWidth, imgHeight, index) {
        console.log("图片宽高", imgWidth, imgHeight)
        // 保持图片比例缩放
        let scale = Math.min(this.canvasWidth / imgWidth, this.canvasHeight / imgHeight);
        this.images[index].canvasWidth = Math.round(imgWidth * scale);
        this.images[index].canvasHeight = Math.round(imgHeight * scale);
        console.log("Canvas宽高", this.images[index].canvasWidth, this.images[index].canvasHeight)
        const ctx = uni.createCanvasContext(this.images[index].canvasId, this);
        // 绘制图片到canvas
        ctx.drawImage(this.images[index].imageUrl, 0, 0, this.images[index].canvasWidth, this.images[index]
          .canvasHeight);
        ctx.save()
        ctx.draw(true, () => {
          // 初始化canvas触摸事件
          this.resetDrawingData();
          // 获取图像数据
          // uni.canvasGetImageData({
          //   canvasId: this.images[index].canvasId,
          //   x: 0,
          //   y: 0,
          //   width: this.images[index].canvasWidth,
          //   height: this.images[index].canvasHeight,
          //   success: (res) => {
          //     let data = res.data;
          //     let r = 1,
          //       g = 1,
          //       b = 1;
          //     // 获取所有像素的累加值
          //     for (let row = 0; row < this.images[index].canvasHeight; row++) {
          //       for (let col = 0; col < this.images[index].canvasWidth; col++) {
          //         if (row == 0) {
          //           r += data[this.images[index].canvasWidth * row + col];
          //           g += data[this.images[index].canvasWidth * row + col + 1];
          //           b += data[this.images[index].canvasWidth * row + col + 2];
          //         } else {
          //           r += data[(this.images[index].canvasWidth * row + col) * 4];
          //           g += data[(this.images[index].canvasWidth * row + col) * 4 + 1];
          //           b += data[(this.images[index].canvasWidth * row + col) * 4 + 2];
          //         }
          //       }
          //     }
          //     // 求rgb平均值
          //     const totalPixels = this.images[index].canvasWidth * this.images[index].canvasHeight;
          //     r = Math.round(r / totalPixels);
          //     g = Math.round(g / totalPixels);
          //     b = Math.round(b / totalPixels);
          //     console.log("rgb", r, g, b)

          //     const rgbSum = r + g + b;
          //     const x = rgbSum / g;
          //     const concentration = this.calculateConcentration(x, this.selectedSolutionType);
          //     this.$set(this.images[index], 'rgbInfo', {
          //       r: r,
          //       g: g,
          //       b: b
          //     });
          //     this.$set(this.images[index], 'concentration', concentration);
          //   },
          //   fail: (err) => {
          //     console.error('获取RGB信息失败', err);
          //   }
          // });
        });
      },
      resetDrawingData() {
        this.isDrawing = false;
        this.startX = 0;
        this.startY = 0;
        this.endX = 0;
        this.endY = 0;
      },
      startDrawing(e, index) {
        const touch = e.touches[0];
        this.startX = touch.x;
        this.startY = touch.y;
        this.isDrawing = true;
      },
      drawRectangle(e, index) {
        if (!this.isDrawing) return;
        const touch = e.touches[0];
        this.endX = touch.x;
        this.endY = touch.y;
        const ctx = uni.createCanvasContext(this.images[index].canvasId, this);
        // 重新绘制图片
        ctx.clearRect(0, 0, this.images[index].canvasWidth, this.images[index].canvasHeight);
        ctx.drawImage(this.images[index].imageUrl, 0, 0, this.images[index].canvasWidth, this.images[index]
          .canvasHeight);
        ctx.setStrokeStyle('#000000');
        ctx.strokeRect(this.startX, this.startY, this.endX - this.startX, this.endY - this.startY);
        ctx.draw(true);
      },
      endDrawing(index) {
        this.isDrawing = false;
        if (this.startX !== this.endX && this.startY !== this.endY) {
          this.calculateRGBInRectangle(index);
        }
      },
      calculateRGBInRectangle(index) {
        const x = Math.min(this.startX, this.endX);
        const y = Math.min(this.startY, this.endY);
        const width = Math.round(Math.abs(this.endX - this.startX));
        const height = Math.round(Math.abs(this.endY - this.startY));
        console.log("width", width)
        console.log("height", height)
        uni.canvasGetImageData({
          canvasId: this.images[index].canvasId,
          x: x,
          y: y,
          width: width,
          height: height,
          success: (res) => {
            let data = res.data;
            let r = 0,
              g = 0,
              b = 0;
            //获取所有像素的累加值
            for (let row = 0; row < height; row++) {
              for (let col = 0; col < width; col++) {
                if (row == 0) {
                  r += data[width * row + col];
                  g += data[width * row + col + 1];
                  b += data[width * row + col + 2];
                } else {
                  r += data[(width * row + col) * 4];
                  g += data[(width * row + col) * 4 + 1];
                  b += data[(width * row + col) * 4 + 2];
                }
              }
            }
            const pixelCount = (width - 2) * (height - 2); // 除去边框的像素数
            r = Math.min(255, Math.max(0, Math.round(r / pixelCount)));
            g = Math.min(255, Math.max(0, Math.round(g / pixelCount)));
            b = Math.min(255, Math.max(0, Math.round(b / pixelCount)));
            // const rgbSum = r + g + b;
            // const xRatio = rgbSum / g;
            const x_input = g / r;
            const concentration = this.calculateConcentration(x_input, this.selectedSolutionType);
            this.$set(this.images[index], 'rgbInfo', {
              r,
              g,
              b
            });
            this.$set(this.images[index], 'concentration', concentration);
          },
          fail: (err) => {
            console.error('Fail getting RGB Info', err);
          }
        });
      },
      calculateConcentration(x, solutionType) {
        console.log("x--->", x)
        let y;
        switch (solutionType) {
          case 'NPA':
            if (x >= 0.615686 && x <= 0.868421) {
              y = 39.993 * x - 25.003;
            } else if (x >= 0.868421 && x <= 1.175115) {
              y = 276.23 * x - 228.93;
            } else if (x >= 1.175115 && x <= 1.268657) {
              y = 104.47 * x - 33.054;
            } else {
              y = 0; // x不在已知范围内
            }
            break;
          case 'ACN':
            if (x >= 0.219608 && x <= 0.893443) {
              y = 14.866 * x - 4.1174;
            } else if (x >= 0.893443 && x <= 1.328125) {
              y = 157.99 * x - 136.41;
            } else if (x >= 1.328125 && x <= 1.344778) {
              y = 1170.4 * x - 1473.7;
            } else {
              y = 0; // x不在已知范围内
            }
            break;
          case 'EtOH':
            if (x >= 0.329412 && x <= 0.704516) {
              y = 27.286 * x - 9.2752;
            } else if (x >= 0.704516 && x <= 1.356878) {
              y = 118.06 * x - 70.057;
            } else if (x >= 1.356878 && x <= 1.37615) {
              y = 382.16 * x - 426.41;
            } else {
              y = 0; // x不在已知范围内
            }
            break;
          case 'IPA':
            if (x >= 0.626016 && x <= 0.978632) {
              y = 27.612 * x - 18.16;
            } else if (x >= 0.978632 && x <= 1.287554) {
              y = 256.7 * x - 237.93;
            } else if (x >= 1.287554 && x <= 1.335287) {
              y = 208.36 * x - 178.42;
            } else {
              y = 0; // x不在已知范围内
            }
            break;
          case 'THF':
            if (x >= 0 && x <= 0.055336) {
              y = 175.32 * x + 0.3653;
            } else if (x >= 0.958549 && x <= 1.023075) {
              y = 790.88 * x - 729.34;
            } else if (x >= 1.220096 && x <= 1.301234) {
              y = 116.26 * x - 51.52;
            } else {
              y = 0; // x不在已知范围内
            }
            break;
          default:
            y = 0;
        }
        return `${y.toFixed(2)}%`;
      },
      deleteImage(index) {
        this.images.splice(index, 1);
      },
      saveImageInfo(image) {
        if (image.rgbInfo) {
          const info =
            `Image URL: ${image.imageUrl}\nR: ${image.rgbInfo.r}\nG: ${image.rgbInfo.g}\nB: ${image.rgbInfo.b}\n浓度: ${image.concentration}`;
          uni.setClipboardData({
            data: info,
            success: () => {
              uni.showToast({
                title: 'Information Saved',
                icon: 'success'
              });
            }
          });
        } else {
          uni.showToast({
            title: 'No RGB Info',
            icon: 'none'
          });
        }
      }
    }
  };
</script>

<style scoped>
  /* 页面容器样式 */
  .container {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    /* 使内容垂直居中 */
    height: 100vh;
    /* 占满视口高度 */
    padding: 20px;
    /* 根据需要调整内边距 */
    box-sizing: border-box;
    /* 让 padding 影响到总宽度和高度 */
    /* 将容器从底部上移到屏幕中央 */
  }

  /* 上传按钮样式 */
  .upload-button {
    margin-bottom: 20px;
    padding: 10px 20px;
    background-color: #007aff;
    color: #fff;
    border-radius: 5px;
    border: none;
    cursor: pointer;
  }

  /* 图片和RGB信息容器样式 */
  .images-container {
    display: flex;
    flex-direction: row;
    flex-wrap: wrap;
    justify-content: center;
    align-items: center;
    width: 100%;
    padding: 20px;
    box-sizing: border-box;
    overflow-y: auto;
    /* Enable vertical scrolling */
    overflow-x: hidden;
    /* Prevent horizontal scrolling */
    max-height: calc(100vh - 100px);
    /* Adjust the max height to prevent overflow */
  }

  /* 单张图片容器样式 */
  .image-wrapper {
    margin-bottom: 10px;
    text-align: center;
    position: relative;
    padding: 10px;
    border: 1px solid #ccc;
    border-radius: 5px;
    max-width: 100%;
    /* Ensure the wrapper doesn't exceed container width */
    display: flex;
    flex-direction: column;
    align-items: center;
    /* Center content horizontally */
  }

  /* 操作按钮容器样式 */
  .action-buttons {
    position: absolute;
    top: 5px;
    right: 5px;
    display: flex;
    flex-direction: column;
    align-items: flex-end;
  }

  /* 操作按钮样式 */
  .action-button {
    margin-bottom: 5px;
    background: none;
    border: none;
    font-size: 20px;
    cursor: pointer;
    color: #333;
    padding: 5px;
    border-radius: 50%;
    width: 35px;
    height: 35px;
    display: flex;
    align-items: center;
    justify-content: center;
    background-color: rgba(255, 255, 255, 0.8);
    box-shadow: 0px 2px 5px rgba(0, 0, 0, 0.2);
  }

  .delete-button {
    color: #ff4d4f;
  }

  .save-button {
    color: #52c41a;
  }

  /* RGB 信息和 Water Fraction 样式 */
  .rgb-info {
    margin-top: 10px;
    font-size: 16px;
    display: flex;
    flex-direction: column;
    align-items: center;
  }

  /* RGB 值上下排列并居中 */
  .rgb-values {
    display: flex;
    justify-content: space-around;
    width: 100%;
    margin-bottom: 5px;
  }

  /* Water Fraction 样式 */
  .fraction-value {
    text-align: center;
    font-size: 18px;
    font-weight: bold;
  }

  /* 选择溶液类型样式 */
  .picker {
    margin: 20px;
    padding: 15px 20px;
    background-color: #ffffff;
    border-radius: 5px;
    text-align: center;
    font-size: 18px;
    color: #ff3503;
    box-shadow: 0 2px 5px rgba(0, 0, 0, 0.15);
  }
</style>
