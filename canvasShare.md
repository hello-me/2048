# 可扩展的canvas绘图工具实现
[示例](https://hello-me.github.io/2048/canvasShare.html)
## requestAnimationFrame API 介绍
requestAnimationFrame是浏览器用于定时循环操作的一个接口，类似于setTimeout，主要用途是按帧对网页进行重绘。
设置这个API的目的是为了让各种网页动画效果（DOM动画、Canvas动画、SVG动画、WebGL动画）能够有一个统一的刷新机制，从而节省系统资源，提高系统性能，改善视觉效果。代码中使用这个API，就是告诉浏览器希望执行一个动画，让浏览器在下一个动画帧安排一次网页重绘。
requestAnimationFrame的优势，在于充分利用显示器的刷新机制，比较节省系统资源。显示器有固定的刷新频率（60Hz或75Hz），也就是说，每秒最多只能重绘60次或75次，requestAnimationFrame的基本思想就是与这个刷新频率保持同步，利用这个刷新频率进行页面重绘。此外，使用这个API，一旦页面不处于浏览器的当前标签，就会自动停止刷新。这就节省了CPU、GPU和电力。
不过有一点需要注意，requestAnimationFrame是在主线程上完成。这意味着，如果主线程非常繁忙，requestAnimationFrame的动画效果会大打折扣。
requestAnimationFrame使用一个回调函数作为参数。这个回调函数会在浏览器重绘之前调用。
```javascript
  window.requestAnimationFrame(callback); 
```
目前，主要浏览器Firefox 23 / IE 10 / Chrome / Safari）都支持这个方法。可以用下面的方法，检查浏览器是否支持这个API。如果不支持，则自行模拟部署该方法。
```javascript
 window.requestAnimFrame = (function(){
  return  window.requestAnimationFrame       || 
          window.webkitRequestAnimationFrame || 
          window.mozRequestAnimationFrame    || 
          window.oRequestAnimationFrame      || 
          window.msRequestAnimationFrame     || 
          function( callback ){
            window.setTimeout(callback, 1000 / 60);
          };
})();
```
上面的setTimeout代码按照1秒钟60次（大约每16.7毫秒一次），来模拟requestAnimationFrame。
使用requestAnimationFrame的时候，只需反复调用它即可。
```javascript
function repeatOften() {
  // Do whatever
  requestAnimationFrame(repeatOften);
}
repeatOften()
```
## 离屏canvas 介绍
canvas的绘制操作，会消耗很多的系统资源，所以当我们有一个复杂的场景时，我们就不应该在每一帧把所有场景元素都重新绘制一遍。
所以，最好是将一些静态的场景元素绘制一次存到缓存中，然后每次绘制从缓存中取数据。
离屏canvas就是用来做这种优化的。
离屏canvas指的就是创建一个不可见的canvas。
可以用如下方式创建一个离屏canvas
```javascript
let outCanvas = document.createElenemt('canvas')
outCanvas.width = 300
outCanvas.height = 300
let outCtx = outCanvas.getContext('2d')
```
像上面这样创建好之后，我们就可以在这个不可见的canvas中绘制我们需要缓存的画面。
然后，在需要的时候我们就可以将他拿出来用了。
使用这个离屏canvas要用到另一个API:
`context.drawImage(img,sx,sy,swidth,sheight,x,y,width,height)`
- img	规定要使用的图像、画布或视频。
- sx	可选。开始剪切的 x 坐标位置。
- sy	可选。开始剪切的 y 坐标位置。
- swidth	可选。被剪切图像的宽度。
- sheight	可选。被剪切图像的高度。
- x	在画布上放置图像的 x 坐标位置。
- y	在画布上放置图像的 y 坐标位置。
- width	可选。要使用的图像的宽度。（伸展或缩小图像）
- height	可选。要使用的图像的高度。（伸展或缩小图像）
示例代码
```javascript
// 获取页面上的canvas元素
let canvas = docuemnt.querySelector('#canvas')
let ctx = cnavas.getContext('2d')
// 将离屏canvas绘制到页面的canvas画布上
ctx.drawImage(outCanvas, 0, 0, width, height)
```
## 代码实现
整个项目分为两大部分
1. 场景
场景负责canvas控制，事件监听，动画处理
2. 精灵
精灵则指的是每一种可以绘制的canvas元素
### sprite实现
#### 父类
```javascript
class Element {
  constructor(options = {
    fillStyle: 'rgba(0,0,0,0)',
    lineWidth: 1,
    strokeStyle: 'rgba(0,0,0,255)'
  }) {
    this.options = options
  }
  setStyle(options){
    this.options =  Object.assign(this.options. options)
  }
}
```
1. 属性：
- options中存储了所有的绘图属性
    - fillStyle：设置或返回用于填充绘画的颜色、渐变或模式
    - strokeStyle：设置或返回用于笔触的颜色、渐变或模式
    - lineWidth：设置或返回当前的线条宽度
    - 使用的都是getContext("2d")对象的原生属性，此处只列出了这三种属性，需要的话还可以继续扩充。
- 有需要可以继续扩充
2. 方法：
- setStyle方法用于重新设置当前精灵的属性
- 有需要可以继续扩充

所有的精灵都继承Element类。
#### 子类
子类就是每一种精灵元素的具体实现，这里我们介绍一遍Circle元素的实现
```javascript
class Circle extends Element {
  // 定位点的坐标（这块就是圆心），半径，配置对象
  constructor(x, y, r = 0, options) {
    // 调用父类的构造函数
    super(options)
    this.x = x
    this.y = y
    this.r = r
  }
  // 改变元素大小
  resize(x, y) {
    this.r = Math.sqrt((this.x - x) ** 2 + (this.y - y) ** 2)
  }
  // 移动元素到新位置，接收两个参数，新的元素位置
  moveTo(x, y) {
    this.x = x
    this.y = y
  }
  // 判断点是否在元素中，接收两个参数，点的坐标
  choose(x, y) {
    return ((x - this.x) ** 2 + (y - this.y) ** 2) < (this.r ** 2)
  }
  // 偏移，计算点和元素定位点的相对偏移量（ofsetX， offsetY）
  getOffset(x, y) {
    return {
      x: x - this.x,
      y: y - this.y
    }
  }
  // 绘制元素实现，接收一个ctx对象，将当前元素绘制到指定画布上
  draw(ctx) {
    // 取到绘制所需属性
    let {
      fillStyle,
      strokeStyle,
      lineWidth
    } = this.options
    // 开始绘制
    ctx.beginPath()
    // 设置属性
    ctx.fillStyle = fillStyle
    ctx.strokeStyle = strokeStyle
    ctx.lineWidth = lineWidth
    // 画圆
    ctx.arc(this.x, this.y, this.r, 0, 2 * Math.PI)
    // 填充颜色
    ctx.stroke()
    ctx.fill()
    // 绘制完成
  }
  // 验证函数，判断当前元素是否满足指定条件，此处用来检验是否将元素添加到场景中。
  validate() {
    return this.r >= 3
  }
}
```
注意事项：
- 构造函数的形参只有两个是必须的，就是定位点的坐标。
- 其它的形参都必须有默认值。

所有方法的调用时机
- 我们在画布上绘制元素的时候回调用resize方法。
- 移动元素的时候调用moveTo方法。
- choose会在鼠标按下时调用，判断当前元素是否被选中。
- getOffset选中元素时调用，判断选中位置。
- draw绘制函数，绘制元素到场景上时调用。

### scene场景的实现
1. 属性介绍
```javascript
class Sence {
  constructor(id, options = {
    width: 600,
    height: 400
  }) {
    // 画布属性
    this.canvas = document.querySelector('#' + id)
    this.canvas.width = options.width
    this.canvas.height = options.height
    this.width = options.width
    this.height = options.height
    // 绘图的对象
    this.ctx = this.canvas.getContext('2d')
    // 离屏canvas
    this.outCanvas = document.createElement('canvas')
    this.outCanvas.width = this.width
    this.outCanvas.height = this.height
    this.outCtx = this.outCanvas.getContext('2d')
    // 画布状态
    this.stateList = {
      drawing: 'drawing',
      moving: 'moving'
    }
    this.state = this.stateList.drawing
    // 鼠标状态
    this.mouseState = {
    // 记录鼠标按下时的偏移量
      offsetX: 0,
      offsetY: 0,
      down: false, //记录鼠标当前状态是否按下
      target: null //当前操作的目标元素
    }
    // 当前选中的精灵构造器
    this.currentSpriteConstructor = null
    // 存储精灵
    let sprites = []
    this.sprites = sprites
    /* .... */
  }
}
```
2. 事件逻辑
```javascript
class Sence {
  constructor(id, options = {
    width: 600,
    height: 400
  }) {
  /* ... */
  // 监听事件
    this.canvas.addEventListener('contextmenu', (e) => {
      console.log(e)
    })
    // 鼠标按下时的处理逻辑
    this.canvas.addEventListener('mousedown', (e) => {
    // 只有左键按下时才会处理鼠标事件
      if (e.button === 0) {
      // 鼠标的位置
        let x = e.offsetX
        let y = e.offsetY
        // 记录鼠标是否按下
        this.mouseState.down = true
        // 创建一个临时target
        // 记录目标元素
        let target = null
        if (this.state === this.stateList.drawing) {
        // 判断当前有没有精灵构造器，有的话就构造一个对应的精灵元素
          if (this.currentSpriteConstructor) {
            target = new this.currentSpriteConstructor(x, y)
          }
        } else if (this.state === this.stateList.moving) {
          let sprites = this.sprites
          // 遍历所有的精灵，调用他们的choose方法，判断有没有被选中
          for (let i = sprites.length - 1; i >= 0; i--) {
            if (sprites[i].choose(x, y)) {
              target = sprites[i]
              break;
            }
          }
          
          // 如果选中的话就调用target的getOffset方法，获取偏移量
          if (target) {
            let offset = target.getOffset(x, y)
            this.mouseState.offsetX = offset.x
            this.mouseState.offsetY = offset.y
          }
        }
        // 存储当前目标元素
        this.mouseState.target = target
        // 在离屏canvas保存除目标元素外的所有元素
        let ctx = this.outCtx
        // 清空离屏canvas
        ctx.clearRect(0, 0, this.width, this.height)
        // 将目标元素外的所有的元素绘制到离屏canvas中
        this.sprites.forEach(item => {
          if (item !== target) {
            item.draw(ctx)
          }
        })
        if(target){
            // 开始动画
            this.anmite()
        }
      }
    })
    this.canvas.addEventListener('mousemove', (e) => {
    //  如果鼠标按下且有目标元素，才执行下面的代码
      if (this.mouseState.down && this.mouseState.target) {
        let x = e.offsetX
        let y = e.offsetY
        if (this.state === this.stateList.drawing) {
        // 调用当前target的resize方法，改变大小
          this.mouseState.target.resize(x, y)
        } else if (this.state === this.stateList.moving) {
        // 取到存储的偏移量
          let {
            offsetX, offsetY
          } = this.mouseState
          // 调用moveTo方法将target移动到新的位置
          this.mouseState.target.moveTo(x - offsetX, y - offsetY)
        }
      }
    })
    document.body.addEventListener('mouseup', (e) => {
      if (this.mouseState.down) {
      // 将鼠标按下状态记录为false
        this.mouseState.down = false
        if (this.state === this.stateList.drawing) {
        // 调用target的validate方法。判断他要不要被加到场景去呢
          if (this.mouseState.target.validate()) {
            this.sprites.push(this.mouseState.target)
          }
        } else if (this.state === this.stateList.moving) {
          // 什么都不做
        }
      }
    })
  }
}
```
3. 方法介绍
```javascript
class Sence {
// 动画
  anmite() {
    requestAnimationFrame(() => {
      // 清除画布
      this.clear()
      // 将离屏canvas绘制到当前canvas上
      this.paint(this.outCanvas)
      // 绘制target
      this.mouseState.target.draw(this.ctx)
      // 鼠标是按下状态就继续执行下一帧动画
      if (this.mouseState.down) {
        this.anmite()
      }
    })
  }
  // 可以将手动的创建的精灵添加到画布中
  append(sprite) {
    this.sprites.push(sprite)
    sprite.draw(this.ctx)
  }
  // 根据ID值，从场景中删除对应元素
  remove(id) {
    this.sprites.splice(id, 1)
  }
  // clearRect清除指定区域的画布内容
  clear() {
    this.ctx.clearRect(0, 0, this.width, this.height)
  }
  // 重绘整个画布的内容
  reset() {
    this.clear()
    this.sprites.forEach(element => {
      element.draw(this.ctx)
    })
  }
  // 将离屏canvas绘制到页面的canvas画布上
  paint(canvas, x = 0, y = 0) {
    this.ctx.drawImage(canvas, x, y, this.width, this.height)
  }
  // 设置当前选中的精灵构造器
  setCurrentSprite(Element) {
    this.currentSpriteConstructor = Element
  }
}
```

