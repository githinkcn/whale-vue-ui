<template>
    <div class="upload">
    </div>
</template>

<script>
    import {CheckFileMd5,FinisUpload} from '@api/file'
    let GUID = WebUploader.Base.guid();
    export default {
        name: 'vue-upload',
        props: {
            accept: {
                type: Object,
                default: null,
            },
            // 上传地址
            url: {
                type: String,
                default: '',
            },
            // 上传最大数量 默认为100
            fileNumLimit: {
                type: Number,
                default: 100,
            },
            // 大小限制 默认2M
            fileSingleSizeLimit: {
                type: Number,
                default: 2048000,
            },
            // 生成formData中文件的key，下面只是个例子，具体哪种形式和后端商议
            keyGenerator: {
                type: Function,
                default(file) {
                    const currentTime = new Date().getTime();
                    const key = `${currentTime}.${file.name}`;
                    return key;
                },
            },
            multiple: {
                type: Boolean,
                default: false,
            },
            // 上传按钮ID
            uploadButton: {
                type: String,
                default: '',
            },
        },
        data() {
            return {
                uploader: null
            };
        },
        mounted() {
            this.initRegisterWebUpload();
            this.initWebUpload();
        },
        methods: {
            initRegisterWebUpload() {
                //注册功能一定要写在前头，否则不会生效
                WebUploader.Uploader.register({
                  //在文件发送之前request，此时还没有分片（如果配置了分片的话），可以用来做文件整体md5验证。在文件发送之前request，此时还没有分片（如果配置了分片的话），可以用来做文件整体md5验证。
                  'before-send-file': 'beforeSendFile',
                  //在分片发送之前request，可以用来做分片验证，如果此分片已经上传成功了，可返回一个rejected promise来跳过此分片上传
                  "before-send": "beforeSend",
                  //在所有分片都上传完毕后，且没有错误后request，用来做分片验证，此时如果promise被reject，当前文件上传会触发错误。
                  "after-send-file": "afterSendFile"
                },{
                    beforeSendFile: function (file) {
                        let $fileStatus = $(`.file-${file.id} .file-status`);
                        let owner = this.owner,
                            deferred = WebUploader.Deferred();
                        owner.md5File(file.source).fail(()=>{
                            deferred.reject();
                        }).progress(function (percent) {
                          
                        }).then(function (md5Value) {
                            let param = {md5:md5Value}
                            CheckFileMd5(param).then(res =>{
                                if(res.isExist){
                                    deferred.reject();
                                    owner.skipFile(file);
                                    $(`.file-${file.id} .progress`).css('width', 100 + '%');
                                    $(`.file-${file.id} .file-status`).html(100 + '%');
                                    $fileStatus.html('上传成功');
                                }else {
                                    deferred.resolve();
                                    file.filehash = md5Value;
                                }
                            }).catch(err=>{
                                console.log(err)
                            })
                        })
                    },
                    beforeSend: function(block){

                    },
                    afterSendFile: function (file) {
                      console.log(file)
                      let chunksTotal = 0;
                      if ((chunksTotal = Math.ceil(file.size / 2048000)) >= 1) {
                        let deferred = WebUploader.Deferred();
                        FinisUpload({
                          id: file.id,
                          name: file.name,
                          chunks: chunksTotal,
                          filehash:file.filehash,
                          size:file.size,
                          type:file.type,
                          ext:file.ext,
                          currPath:''
                        }).then(res=>{
                          deferred.resolve();
                        }).catch(err=>{
                          deferred.reject();
                        });
                        return deferred.promise();
                      }
                    }
                });
            },
            initWebUpload() {
                let that = this;
                this.uploader = WebUploader.create({
                    auto: true, // 选完文件后，是否自动上传
                    swf: '/webuploader-0.1.5/Uploader.swf',  // swf文件路径
                    server: this.url,  // 文件接收服务端
                    pick: {
                        id: this.uploadButton,     // 选择文件的按钮
                        multiple: this.multiple,   // 是否多文件上传 默认false
                        label: '',
                    },
                    accept: this.getAccept(this.accept),  // 允许选择文件格式。
                    threads: 5,
                    fileNumLimit: this.fileNumLimit, // 限制上传个数
                    //fileSingleSizeLimit: this.fileSingleSizeLimit, // 限制单个上传图片的大小
                    chunked: true,          //分片上传
                    chunkSize: 2048000,    //分片大小
                    duplicate: false,  // 重复上传
                });
                // 当有文件被添加进队列的时候，添加到页面预览
                this.uploader.on('fileQueued', (file) => {
                    this.$emit('fileChange', file);
                });
                this.uploader.on('uploadStart', (file) => {
                    // 在这里可以准备好formData的数据
                    //this.uploader.options.formData.key = this.keyGenerator(file);
                });
                // 文件上传过程中创建进度条实时显示。
                this.uploader.on('uploadProgress', (file, percentage) => {
                    this.$emit('progress', file, percentage);
                });
                this.uploader.on('uploadSuccess', (file, response) => {
                    this.$emit('success', file, response);
                });
                this.uploader.on('uploadAccept', (object, ret) => {
                  switch (ret.code) {
                    case 401:
                      this.uploader.cancelFile(object.file);
                      that.$router.push("/login")
                      break;
                    default:
                  }
                });
                this.uploader.on('uploadError', (file, reason) => {
                    console.error(reason);
                    this.$emit('uploadError', file, reason);
                });
                this.uploader.on('uploadBeforeSend', (object ,data, headers) => {
                    $.extend(headers, {
                        "Authorization": 'Bearer '+that.$utils.cookies.get('token')
                    });
                    $.extend(data,{
                        "guid": GUID,
                        "dir": "",
                        "ext": data.type
                    });
                });
                this.uploader.on('error', (type) => {
                    let errorMessage = '';
                    if (type === 'F_EXCEED_SIZE') {
                        errorMessage = `文件大小不能超过${this.fileSingleSizeLimit / (1024 * 1000)}M`;
                    } else if (type === 'Q_EXCEED_NUM_LIMIT') {
                        errorMessage = '文件上传已达到最大上限数';
                    } else {
                        errorMessage = `上传出错！请检查后重新上传！错误代码${type}`;
                    }
                  this.$message.error(errorMessage);
                });
                this.uploader.on('uploadComplete', (file, response) => {
                    this.$emit('complete', file, response);
                });
            },
            upload() {
                this.uploader.upload();
            },
            stop() {
              this.uploader.stop(true);
            },
            // 取消并中断文件上传
            cancelFile(file) {
                this.uploader.cancelFile(file);
            },
            // 在队列中移除文件
            removeFile(file, bool) {
                this.uploader.removeFile(file, bool);
            },
            getStats() {
                return this.uploader.getStats();
            },
            getFromData() {

            },
            getAccept(accept) {
                switch (accept) {
                    case 'text':
                        return {
                            title: 'Texts',
                            exteensions: 'doc,docx,xls,xlsx,ppt,pptx,pdf,txt',
                            mimeTypes: '.doc,docx,.xls,.xlsx,.ppt,.pptx,.pdf,.txt'
                        };
                        break;
                    case 'video':
                        return {
                            title: 'Videos',
                            exteensions: 'mp4',
                            mimeTypes: '.mp4'
                        };
                        break;
                    case 'image':
                        return {
                            title: 'Images',
                            exteensions: 'gif,jpg,jpeg,bmp,png',
                            mimeTypes: '.gif,.jpg,.jpeg,.bmp,.png'
                        };
                        break;
                    default: return accept
                }
            },
        },
    };
</script>

<style lang="scss">
    .webuploader-container {
        position: relative;
    }
    .webuploader-element-invisible {
        position: absolute !important;
        clip: rect(1px 1px 1px 1px); /* IE6, IE7 */
        clip: rect(1px,1px,1px,1px);
    }
    .webuploader-pick {
        position: relative;
        display: inline-block;
        cursor: pointer;
        background: none;
        padding: 0px;
        color: #fff;
        text-align: center;
        border-radius: 3px;
        overflow: hidden;
    }
    .webuploader-pick-hover {
        background: none;
    }
    .webuploader-pick-disable {
        opacity: 0.6;
        pointer-events:none;
    }
</style>
