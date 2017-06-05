<template>
  <v-touch ref="pinch" tag="div"
           @panmove="panMove"
           @panstart="panStart"
           @pinchin="pinchIn"
           @pinchout="pinchOut"
           @pinchend="pinchEnd"
           @pinchstart="pinchStart"
           :pinchstart-options="{threshold:10}" id="container"
           style="width:100%;height:100%;display:-webkit-box;-webkit-box-align: center;-webkit-box-pack:center;">
    <div class="zzc" id="overlay"></div>
    <canvas  id="canvas" :src="url"></canvas>
    <input type="file" name="file" id="file" style="display:none" multiple @change="readFile"/>
    <div class="imgEditorBtn">
      <button @click="leftRotation">左旋</button>
      <button @click="rightRotation">右旋</button>
      <button @click="chooseImage">选择文件</button>
      <button @click="crop">确定</button>
    </div>
  </v-touch>
</template>
<style type="text/css">
  .imgEditorBtn{position: absolute;bottom: 0;left: 0;}
</style>
<script>
    import qiniu from '../../js/qiniu';
    import constants from '../../js/constants';
    import {Toast} from 'mint-ui';
    export default {
        data(){
            return {
                currentScale: 1,
                lastScale: 1,
                num: 0,
                h: 0,
                w: 0,
                dom: null,
                img: null,
                startx: 0,
                starty: 0,
                x: 0,
                y: 0,
                /**
                 *旋转角度
                 */
                deg: 0,

                /**
                 * imgView的使用模式
                 */
                model: '',

                /**
                 * 编辑按钮文本
                 */
                editText: '编辑',

                /**
                 * 遮罩层的dom节点
                 */
                overlay: '',
                /**
                 * 父容器的dom节点
                 */
                container: '',
                /**
                 * 画布的dom节点
                 */
                canvas:null,
                /**
                 * 画布的上下文对象
                 */
                context:null,
                /**
                 * 截图后产生的dataUrl
                 */
                imgDataUrl:'',
                /**
                 * 截图产生的图片size
                 */
                imgDataSize:'',
                /**
                 *  可选择图片的大小
                 */
                maxSize: {
                    type: Number,
                    default: 8*1024*1024 //上传大小限制
                },
                // 读入的图片
                image:null,
                /**
                 * 是否以确认裁剪
                 */
                isCrop:false,
                /**
                 * 旋转次数,用来判断旋转的方向
                 */
                rotationCount:0,
            }
        },
        computed: {
            transform: {
                get: function () {
                    return {

                    }
                }
            },
            url: function () {
                return this.$route.query.imgUrl;
            }
        },
        mounted(){
            // 获取元素节点
            this.overlay = document.getElementById('overlay');
            this.container = document.getElementById("container");
            this.canvas=document.getElementById('canvas');
            this.context=this.canvas.getContext('2d');
        },
        methods: {
            /**
             * 选择文件
             */
            chooseImage () {
                // 当前浏览器是否兼容文件系统
                if (window.File && window.FileReader && window.FileList && window.Blob) {
                    // 打开文件选择器
                    document.getElementById('file').click();
                } else {
                    alert('The File APIs are not fully supported in this browser.');
                }
            },
            /**
             *
             */
            calcCanvas(){
                this.context.clearRect(0, 0, this.canvas.width, this.canvas.height);
                let clientWidth=document.documentElement.clientWidth;
                let clientHeight=document.documentElement.clientHeight;
                let imgWidth=this.image.width;
                let imgHeight=this.image.height;
                if(imgWidth>clientWidth||imgHeight>clientHeight){
                    // 设置画板的最大宽高为客户端宽高
                    this.canvas.width = clientWidth;
                    this.canvas.height = clientHeight;
                    // 图片超过画板,将被drawImage缩放,因此定位应该为左上角原点
                    this.canvas.style.left= 0+'px';
                    this.canvas.style.top= 0+'px';
                    this.x=0;
                    this.y= 0;
                    // 画出缩放图片
                    this.context.drawImage(this.image, 0, 0, this.image.width, this.image.height,0,0,clientWidth,clientHeight);
                }else{
                    // 设置画板的最大宽高为图片宽高
                    this.canvas.width = imgWidth;
                    this.canvas.height = imgHeight;
                    // 图片超过画板,不会被drawImage缩放,因此定位应该为屏幕中心
                    this.canvas.style.left= (document.documentElement.clientWidth-this.image.width)/2+'px';
                    this.canvas.style.top= (document.documentElement.clientHeight-this.image.height)/2+'px';
                    // 设定起始偏移量
                    this.x=(document.documentElement.clientWidth-this.image.width)/2;
                    this.y= (document.documentElement.clientHeight-this.image.height)/2;
                    this.context.drawImage(this.image, 0, 0, this.image.width, this.image.height,0,0,imgWidth,imgHeight);
                }
            },
            /**
             * 读取的图片流准备渲染到画板上
             * @param src 需要接收fileReader.result
             */
            render(src){
                this.image = new Image();
                // 先指定onload方法,再赋值src,否则onload不会调用
                this.image.onload = ()=>{
                    this.calcCanvas();
                };
                this.image.src = src;
            },
            /**
             * 读取文件
             * @param event 文件读取事件
             */
            readFile(event){
                let file = event.target.files[0];
                if (!file) {
                    return;
                }
                if (this.maxSize) {
                    if(file.size>this.maxSize.default){
                        Toast({message:'图片大小限定在8M以内,请重新选择图片上传.'});
                        return;
                    }
                }
                // 微信浏览器中file对象无法访问到任何数据,所以无法像谷歌那样判断文件类型,但是用FileReader可以加载出来
//        // 检验是否为图像文件
//        if(!/image\/\w+/.test(file.type)){
////          Toast({message:'请选择支持的图片文件',position:'bottom'});
//          return false;
//        }
                var reader = new FileReader();
                // 将文件以Data URL形式读入页面
                reader.readAsDataURL(file);
                reader.onload=e=>{
                    // 显示图片
                    this.render(reader.result);
                }
                // 关闭禁用
                this.isCrop=false;
            },
            upload:function(){
                // 获取Token值
                qiniu.getToken(constants.qiniu.userScene).then(res => {
                    let token = res.token;
                let domain = res.domain;
                qiniu.uploadBas64(this.imgDataUrl,-1,token).then(res=>{
                    let url=domain+res.data.key;
                console.log("ok",res);
                console.log("domian",domain,res.data.key)
            },res=>{
                    console.err("error:",res);
                });
            }, res => {
                    Toast({message: "获取token失败.", position: 'bottom'});
                });
            },
            /**
             * 确认裁剪
             */
            crop: function () {
                if(!this.image){
                    Toast({message:'请先选择一张图片',position:'bottom'})
                    return;
                }
                let left = 0,top = 0;
                // 计算截取的起点
                left=this.overlay.offsetLeft-this.x;
                top=this.overlay.offsetTop-this.y;
                // 获取到要截取的imgData
                let imgData=this.context.getImageData(left,top,200,200);
                this.canvas.width=200;
                this.canvas.height=200;
                // 移动canvas到overlay层
                this.x=this.overlay.offsetLeft;
                this.y=this.overlay.offsetTop;
                this.canvas.style.left=(this.x+2)+'px';
                this.canvas.style.top=(this.y+2)+'px';
                // 显示截图
                this.context.putImageData(imgData,0,0);
                // 保存base64编码
                this.imgDataUrl=this.canvas.toDataURL('image/PNG');
                // 保存当前绘图状态
                // 准备上传
                this.upload();
                this.isCrop=true;
            },
            /**
             * 左旋
             */
            leftRotation: function () {
                if(this.isCrop)return;
                if(!this.image)return;
                this.deg=this.deg-90;
                this.rotationCount= this.rotationCount-1;
                let clientWidth=document.documentElement.clientWidth;
                let clientHeight=document.documentElement.clientHeight;
                let imgWidth=this.image.width;
                let imgHeight=this.image.height;
                if(imgWidth>clientWidth||imgHeight>clientHeight){
                    // 设置画板的最大宽高为客户端宽高
                    if(Math.abs(this.rotationCount%4)==0||Math.abs(this.rotationCount%4)==2){// 旋转在正中心
                        this.canvas.width = clientWidth;
                        this.canvas.height = clientHeight;
                        // 图片超过画板,将被drawImage缩放,因此定位应该为左上角原点
                        this.canvas.style.left= 0+'px';
                        this.canvas.style.top= 0+'px';
                        this.x=0;
                        this.y= 0;
                    }else{
                        this.canvas.width = clientHeight;
                        this.canvas.height = clientWidth;
                        // 图片超过画板,将被drawImage缩放,因此定位应该为左上角原点
                        this.canvas.style.left= (clientWidth-this.canvas.width)/2+'px';
                        this.canvas.style.top= (clientHeight-this.canvas.height)/2+'px';
                        this.x= (clientWidth-this.canvas.width)/2;
                        this.y=(clientHeight-this.canvas.height)/2;
                    }
                    // 计算旋转原点
                    let x = this.canvas.width/2; // 画布宽度的一半
                    let y = this.canvas.height/2;// 画布高度的一半
                    this.context.clearRect(0,0, this.canvas.width, this.canvas.height);// 先清掉画布上的内容
                    this.context.translate(x,y);// 将绘图原点移到画布中点
                    this.context.rotate(this.deg*Math.PI/180);// 旋转角度
                    this.context.translate(-x,-y);// 将画布原点移动
                    // 画出缩放图片
                    if(Math.abs(this.rotationCount%4)==0||Math.abs(this.rotationCount%4)==2){// 旋转在正中心
                        this.context.drawImage(this.image, 0, 0, this.image.width, this.image.height,0,0,clientWidth,clientHeight);
                    }else{
                        this.context.drawImage(this.image, 0, 0, this.image.width, this.image.height,0,0,clientHeight,clientWidth);
                    }
                }else{
                    // 设置画板的最大宽高为图片宽高
                    this.canvas.width = imgWidth;
                    this.canvas.height = imgHeight;
                    // 图片超过画板,不会被drawImage缩放,因此定位应该为屏幕中心
                    this.canvas.style.left= (document.documentElement.clientWidth-this.image.width)/2+'px';
                    this.canvas.style.top= (document.documentElement.clientHeight-this.image.height)/2+'px';
                    // 设定起始偏移量
                    this.x=(document.documentElement.clientWidth-this.image.width)/2;
                    this.y= (document.documentElement.clientHeight-this.image.height)/2;
                    // 计算旋转原点
                    let x = this.canvas.width/2; // 画布宽度的一半
                    let y = this.canvas.height/2;// 画布高度的一半
                    this.context.clearRect(0,0, this.canvas.width, this.canvas.height);// 先清掉画布上的内容
                    this.context.translate(x,y);// 将绘图原点移到画布中点
                    this.context.rotate(this.deg*Math.PI/180);// 旋转角度
                    this.context.translate(-x,-y);// 将画布原点移动
                    // 画出缩放图片
                    this.context.drawImage(this.image, 0, 0, this.image.width, this.image.height,0,0,imgWidth,imgHeight);
                }
            },
            /**
             * 右旋
             */
            rightRotation: function () {
                if(this.isCrop)return;
                if(!this.image)return;
                this.deg=this.deg+90;
                this.rotationCount= this.rotationCount+1;
                let clientWidth=document.documentElement.clientWidth;
                let clientHeight=document.documentElement.clientHeight;
                let imgWidth=this.image.width;
                let imgHeight=this.image.height;
                if(imgWidth>clientWidth||imgHeight>clientHeight){
                    // 设置画板的最大宽高为客户端宽高
                    if(Math.abs(this.rotationCount%4)==0||Math.abs(this.rotationCount%4)==2){// 旋转在正中心
                        this.canvas.width = clientWidth;
                        this.canvas.height = clientHeight;
                        // 图片超过画板,将被drawImage缩放,因此定位应该为左上角原点
                        this.canvas.style.left= 0+'px';
                        this.canvas.style.top= 0+'px';
                        this.x=0;
                        this.y= 0;
                    }else{
                        this.canvas.width = clientHeight;
                        this.canvas.height = clientWidth;
                        // 图片超过画板,将被drawImage缩放,因此定位应该为左上角原点
                        this.canvas.style.left= (clientWidth-this.canvas.width)/2+'px';
                        this.canvas.style.top= (clientHeight-this.canvas.height)/2+'px';
                        this.x= (clientWidth-this.canvas.width)/2;
                        this.y=(clientHeight-this.canvas.height)/2;
                    }
                    // 计算旋转原点
                    let x = this.canvas.width/2; // 画布宽度的一半
                    let y = this.canvas.height/2;// 画布高度的一半
                    this.context.clearRect(0,0, this.canvas.width, this.canvas.height);// 先清掉画布上的内容
                    this.context.translate(x,y);// 将绘图原点移到画布中点
                    this.context.rotate(this.deg*Math.PI/180);// 旋转角度
                    this.context.translate(-x,-y);// 将画布原点移动
                    // 画出缩放图片
                    if(Math.abs(this.rotationCount%4)==0||Math.abs(this.rotationCount%4)==2){// 旋转在正中心
                        this.context.drawImage(this.image, 0, 0, this.image.width, this.image.height,0,0,clientWidth,clientHeight);
                    }else{
                        this.context.drawImage(this.image, 0, 0, this.image.width, this.image.height,0,0,clientHeight,clientWidth);
                    }
                }else{
                    // 设置画板的最大宽高为图片宽高
                    this.canvas.width = imgWidth;
                    this.canvas.height = imgHeight;
                    // 图片超过画板,不会被drawImage缩放,因此定位应该为屏幕中心
                    this.canvas.style.left= (document.documentElement.clientWidth-this.image.width)/2+'px';
                    this.canvas.style.top= (document.documentElement.clientHeight-this.image.height)/2+'px';
                    // 设定起始偏移量
                    this.x=(document.documentElement.clientWidth-this.image.width)/2;
                    this.y= (document.documentElement.clientHeight-this.image.height)/2;
                    // 计算旋转原点
                    let x = this.canvas.width/2; // 画布宽度的一半
                    let y = this.canvas.height/2;// 画布高度的一半
                    this.context.clearRect(0,0, this.canvas.width, this.canvas.height);// 先清掉画布上的内容
                    this.context.translate(x,y);// 将绘图原点移到画布中点
                    this.context.rotate(this.deg*Math.PI/180);// 旋转角度
                    this.context.translate(-x,-y);// 将画布原点移动
                    // 画出缩放图片
                    this.context.drawImage(this.image, 0, 0, this.image.width, this.image.height,0,0,imgWidth,imgHeight);
                }
//        this.context.drawImage(this.image,0,0);//绘制图片
            },
            // 滑动开始
            panStart(event) {
                if(this.isCrop)return;
                this.startx = this.x;
                this.starty = this.y;
            },
            // 滑动
            panMove(event) {
                if(this.isCrop)return;
                if(!this.image)return;
                this.x = this.startx + event.deltaX;
                this.y = this.starty + event.deltaY;
                this.canvas.style.left=this.x+"px";
                this.canvas.style.top=this.y+"px";
            },
            // 双击触摸
            doubleTap(event) {
                if(this.isCrop)return;
                if (this.currentScale < 2) {
                    this.currentScale = 2;
                    this.img.style.transform = "scale(" + 2.0 + ")";
                } else if (this.currentScale == 2) {
                    this.currentScale = 1;
                    this.img.style.transform = "scale(" + 1.0 + ")";
                }
            }
            ,
            // 捏 动作开始
            pinchStart(event) {
                if(this.isCrop)return;
                this.lastScale = this.currentScale;
            }
            ,
            // 捏 动作结束
            pinchEnd(event) {
                if(this.isCrop)return;
                this.lastScale = this.currentScale;
                this.num = 0;
            },
            pinchIn(event) {// 捏合是没有问题的
                if(this.isCrop)return;
                // 防抖动
                if (this.num < 3) {
                    this.num = this.num + 1;
                    return;
                }
                // 防止图片过小
                if (this.currentScale <= 0.5) {
                    return;
                } else {
                    // 设置当前缩放比例
                    this.currentScale = event.scale * this.lastScale;
                    this.context.scale(this.currentScale);
                }
            },
            pinchOut: function (event) {// 捏开
                if(this.isCrop)return;
                // 防抖动
                if (this.num < 3) {
                    this.num = this.num + 1;
                    return;
                }
                // 防止图片过大
                if (this.currentScale >= 2) {
                    return;
                } else {
                    // 设置当前缩放比例
                    this.currentScale = event.scale * this.lastScale;
                    this.context.scale(this.currentScale);
                }
            }
        }
    }
</script>
<style>
  /** {
    margin: 0;
    padding: 0;
  }*/

  html, body {
    width: 100%;
    height: 100%;
  }
  #canvas{
    position: absolute;
    left:0px;
    top:0px;
    /*maxWidth: 100%;*/
    /*maxHeight: 100%;*/
  }
  .zzc {
    position: absolute;
    left: 50%;
    top: 50%;
    margin-left: -100px;
    margin-top: -100px;
    z-index: 999;
    width: 200px;
    height: 200px;
    border: solid 2px rgba(222, 60, 80, .9);
    box-sizing: content-box;
    pointer-events: none;
  }
</style>
