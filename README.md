#index.js
```js
publish:{
      remoteDir:'/opt/dss/client/test',//远程上传目录(服务器)
      localSourcePath:"",//本地需要上传的资源路径
      serverConfig:{//服务器验证信息
          host: '',
          username: 'root',
          password: '7ujMko0admin123',
          algorithms: {
              "kex": [
                "diffie-hellman-group1-sha1",
                "ecdh-sha2-nistp256",
                "ecdh-sha2-nistp384",
                "ecdh-sha2-nistp521",
                "diffie-hellman-group-exchange-sha256",
                "diffie-hellman-group14-sha1"
              ],
              "cipher": [
                "3des-cbc",
                "aes128-cbc",
                "aes192-cbc",
                "aes256-cbc",
                "aes128-ctr",
                "aes192-ctr",
                "aes256-ctr",
                "aes128-gcm@openssh.com",
                "aes256-gcm@openssh.com",
                "arcfour",
                "arcfour128",
                "arcfour256",
                "blowfish-cbc",
                "cast128-cbc",
              ],
              "serverHostKey": [
                "ssh-rsa",
                "ecdsa-sha2-nistp256",
                "ecdsa-sha2-nistp384",
                "ecdsa-sha2-nistp521"
              ],
              "hmac": [
                "hmac-sha2-256",
                "hmac-sha2-512",
                "hmac-sha1"
              ]
          }
      }
```
#ssh.js
```js
/**
 * 通过ssh sftp 远程上传文件
 * author yangyangwang
 * 2018-08-29
 */
const zipper = require('zip-local');
const fs = require("fs");
const chalk = require("chalk");
const ora = require("ora");
const path = require("path");
let Client = require('ssh2').Client;
class uploadBySftp{
    constructor(serverConfig){
        this.config = serverConfig;
        this.client = null;
        this.spanner = ora("");
        this.sftp = null;
        this.newName = null;
        this.remotePath = null;
    };
    close(){
       if( this.client){
        this.client.end();
       }
    };
    async connect(){
        return new Promise((resolve,reject)=>{
            let conn = new Client()
                conn.on("ready",()=>{
                     this.client = conn;
                     this.spanner.succeed(chalk.green("ssh连接成功"));
                     resolve(this.client);
                })
                conn.connect(this.config);
                conn.on("error",(err)=>{
                    console.error(chalk.red("ssh连接失败"));
                    reject("ssh连接失败");
                })
        })

    };
    testCommand(){
        console.log("测试执行远程命令")
        let command="unzip -o /wang/dist.zip -d /wang";
        this.client.exec(command,(err,stream)=>{
            if (err) throw err;
            stream.on('close', function(code, signal) {
                this.client.end();
              }).on('data', function(data) {
                console.log('STDOUT: ' + data);
              }).stderr.on('data', function(data) {
                console.log('STDERR: ' + data);
              });
        })
    };
    /**
     * 回滚上一个版本
     */
    rolUp(){
       let desc =  "******************************************\n"
                  +"***             开始回滚               ***\n"
                  +"******************************************\n"
       console.log(chalk.yellow(desc))
       return new Promise(async (resolve,reject)=>{
            //如果有信名字生成代表已经将原来的问价夹重命名了,所以先判断
            if(!this.newName){
                this.spanner.succeed(chalk.green(".回滚完成"))
                this.spanner.stop();
                resolve(true)
            }else{
              this.client.sftp(async (err,sftp)=>{
                    if(err){
                      reject (err);
                    }
                    if(this.newName){
                      let exist =  await this.fileExistAsync(sftp,this.remotePath);
                      if(exist){
                          //先删除新生成的文件
                        await this.deleteFolder(this.remotePath,1000)
                          //将原来已经重新命名的文件夹还原回来
                        await this.renameAsync(sftp,this.newName,this.remotePath);
                        this.spanner.succeed(chalk.green(".回滚完成"))
                      }
                    }
                    this.spanner.stop()
                    resolve(true)
              });
            }
       })
    };

    /**
     * 同步压缩需要上传的本地文件夹或者文件
     */
    static async zipLocalFile(loaclPath,targetPath){
        return new Promise((resolve,reject)=>{
            //判断源文件是否存在
            let spanner = ora(chalk.green("开始压缩本地文件..."));
            let loaclPathExist = fs.existsSync(loaclPath);
            if(!loaclPathExist){
                spanner.fail(chalk.red(".压缩的目标文件不存在"));
                reject("压缩的目标文件不存在")
                return;
            }
            let baseName = path.basename(loaclPath);
                targetPath = path.join(loaclPath,`${baseName}.zip`)
            let targetPathExist = fs.existsSync(targetPath);
            //判断目标文件时候存在 存在就删除
            if(targetPathExist){
                spanner.succeed(chalk.green(`.删除本地已经存在的压缩文件`));
                fs.unlinkSync(targetPath);
            }
            spanner.start();
            //开始压缩本地文件
            zipper.sync.zip(loaclPath).compress().save(targetPath);
            spanner.succeed(chalk.green("本地文件压缩成功"))
            spanner.stop();
            resolve(targetPath);
        })
    };
    /**
     * 同步解压远程文件到目标文件夹
     * @param {*远程压缩文件路径} remotePath
     * @param {*解压到的远程目录路径} remoteTargetPath
     * @param {*是否显示解压日志} showLog
     */
    unzipRemoteFile(remotePath,remoteTargetPath,showLog){
        let _self = this;
        let command=`unzip -o ${remotePath} -d ${remoteTargetPath}`;
        let desc=  "开始解压远程压缩文件"+remotePath;
        let spanner = ora(chalk.green(desc));
        spanner.start();
        return new Promise((resolve,reject)=>{
            this.client.exec(command,(err,stream)=>{
                if (err){
                    console.log("error:",err)
                    reject(err)
                };
                spanner.stop();
                stream.on('close', function(code, signal) {
                      spanner.succeed(chalk.green(`[${remotePath}]解压成功`))
                     resolve(remotePath)
                  }).on('data', function(data) {
                      if(showLog){
                        spanner.info(data);
                      }
                    //console.log('exec-->' + data);
                  }).stderr.on('data', function(data) {
                	  spanner.fail(chalk.red(`[${remotePath}]解压失败`))
                       reject(false)
                  });
            })
        })
    };
   /**
    * 同步删除远程zip文件
    * @param {*远程资源路径} remotePath
    */
    deleteZipFile(remotePath){
    	let desc=  "开始删除远程压缩文件"+remotePath;
        this.spanner = ora(chalk.green(desc))
        this.spanner.start();
        return new Promise((resolve,reject)=>{
            this.client.sftp(async (err,sftp)=>{
                await this.unlinkSync(sftp,remotePath);
                setTimeout(()=>{
                  this.spanner.stop();
                },500)
                resolve(true);
            })
        })
    };
    /**
     * 同步开始上传本地资源到远程服务器上
     * @param {*本地资源路径} localPath
     * @param {*远程资源路径} remotePath
     * 
     */
    startUploadFile(localPath,remotePath){
        return new Promise((resolve,reject)=>{
            this.client.sftp(async (err,sftp)=>{
                this.sftp = sftp;
                let fileStat = fs.statSync(localPath);
                //判断远程文件有没有存在
                try {
                    let exist =  await this.fileExistAsync(sftp,remotePath);
                    if(exist){
                        let newName = `${remotePath}${new Date().getTime()}`;
                        this.newName = newName;
                        this.remotePath = remotePath;
                        await this.renameAsync(sftp,remotePath,newName)
                    }
                    await this.mkdirAsync(sftp,remotePath);
                    if(fileStat.isFile()){
                    	let spanner = ora(chalk.green("开始上传文件..."));
                        spanner.start()
                        remotePath = path.join(remotePath,path.basename(localPath)).split(path.sep).join("/");
                        await this.uploadFileAsync(sftp,localPath,remotePath);
                        spanner.stop()
                     }else{
                        this.spanner.warn(chalk.red(`.目前只支持普通上传文件`))
                        uploadFlag = false;
                     }
                     resolve(remotePath)
                } catch (error) {
                     console.log("error:",error);
                     spanner.stop()
                     resolve(false)
                }
            })
        })

    };
    /**
     *同步删除远程文件
     * @param {*sftp} sftp
     * @param {*需要删除的远程文件地址} remotePath
     */
    unlinkSync(sftp,remotePath){
        return new Promise((resolve,reject)=>{
            sftp.unlink(remotePath,(err)=>{
                if(err){
                    this.spanner.fail(chalk.red(`文件[${remotePath}]删除失败`))
                    reject(err)
                }
                this.spanner.succeed(chalk.green(`文件[${remotePath}]删除成功`))
                resolve(true)
            })
        })
    };
    /**
     * 同步删除远程文件夹
     * @param {*sftp} sftp
     * @param {*远程文件夹地址} remotePath
     */
    deleteFolder(remotePath,time){
      if(!remotePath){
          return;
      }
      return new Promise((resolve,reject)=>{
        let command = `rm -rf ${remotePath}`
        this.client.exec(command,(err,stream)=>{
            if(err){
              this.spanner.fail(chalk.red(`.文件夹[${remotePath}删除失败`))
              reject(err)
            }
            this.spanner.succeed(chalk.green(`.文件夹[${remotePath}]删除成功`))
            setTimeout(()=>{
              resolve(true)
            },time||0)

        });
     })
    }
    /**
     *同步重命名远程文件或文件
     * @param {*命名前名称} oldName
     * @param {*命名后名称} newName
     */
    renameAsync(sftp,oldName,newName){
        return new Promise((resolve,reject)=>{
            sftp.rename(oldName,newName,(err)=>{
                if(err){
                     console.log(err)
                    this.spanner.fail(chalk.red(`${oldName}文件重命名失败:${newName}`))
                    resolve(err);
                }else{
                    this.spanner.succeed(chalk.green(`${oldName}文件重命名成功:${newName}`))
                    resolve(true);
                }
            })
        })
    };

    /**
     * 同步判断远程主机上是否已经存在该文件或文件夹
     * @param {*远程文件地址} url
     */
    fileExistAsync(sftp,url){
        return new Promise((resolve,reject)=>{
            sftp.exists(url,(flag)=>{
                resolve(flag)
            })
        })
    };

   /**
    * 远程主机上同步创建文件夹
    * @param {*远程文件地址} url
    */
    mkdirAsync(sftp,url){
        return new Promise((resolve,reject)=>{
            sftp.mkdir(url,{},(err)=>{
                if(err){
                    this.spanner.fail(chalk.red(`.文件夹[${url}创建失败`))
                    reject(err)
                }else{
                    this.spanner.succeed(chalk.green(`.文件夹[${url}]创建成功`))
                    resolve(true)
                }
            })
        })
   };

   /**
    * 同步上传文件
    * @param {*本地资源地址} source
    * @param {*远程资源地址} remote
    */
    uploadFileAsync(sftp,source,remote){
        //let sftp = this.sftp;
        return new Promise((resolve,reject)=>{
            let stat = fs.statSync(source);
            sftp.fastPut(source,remote,{chunkSize:stat.size},(err)=>{
                if(err){
                    this.spanner.stop();
                    this.spanner.fail(chalk.red("."+source+"文件上传失败"))
                    reject(err)
                }else{
                    this.spanner.stop();
                    this.spanner.succeed(chalk.green("."+remote+"文件上传成功"))
                    resolve(remote)
                }
            })
        })

   }

}

module.exports = uploadBySftp;

```
#publish
```js
const uploadBySftp = require("./modules/ssh.js");
let config = require("../config/index.js");
const path = require("path");
const chalk = require("chalk");
const ora = require("ora");
async function startUpload(config){
      config = initOption(config);
      const localPath = config.publish.localSourcePath;
      const targetPath = config.publish.remoteDir;
      let desc =  "*******************************************\n"
                 +"***              开始部署               ***\n"
                 +"*******************************************\n"
    console.log(chalk.yellow(desc))
    console.log("remote server ip:",config.publish.serverConfig.host)
    console.log("remote Path:",config.publish.remoteDir,"\n")
    //1 压缩
    let localZipPath = await uploadBySftp.zipLocalFile(localPath,targetPath);
    //2 连接ssh
     let upload = new uploadBySftp(config.publish.serverConfig);
     await upload.connect();
     // upload.testCommand();
     try {
            //3 上传文件
            let remotePath = await upload.startUploadFile(localZipPath,targetPath,true);
            //4 解压
            let remoteZipPath = await upload.unzipRemoteFile(remotePath,targetPath,false);
            //5 删除远程压缩文件
            await upload.deleteZipFile(remoteZipPath);
            //删除原先已经存在的文件夹
            await upload.deleteFolder(upload.newName);
            let desc =  "\n******************************************\n"
                       +"***              部署成功              ***\n"
                       +"******************************************\n"
            console.log(chalk.green(desc));
     
     } catch (error) {
            console.error(chalk.red("文件部署失败"));
            await upload.rolUp();
     }
     //关闭连接
     upload.close();
}
/**
 * 初始化参数
 */
function initOption(config){
   let hostPattern = /(?:(?:(?:25[0-5]|2[0-4]\d|(?:(?:1\d{2})|(?:[1-9]?\d)))\.){3}(?:25[0-5]|2[0-4]\d|(?:(?:1\d{2})|(?:[1-9]?\d))))/ig;
   let remoteHost = process.argv[2];
   let dirPattern = /(\/opt\/dss\/client\/){1}/ig
   let dir = process.argv[3];
  if(remoteHost){
    if(hostPattern.test(remoteHost)){
      config.publish.serverConfig.host = remoteHost;
    }
  }

   if(dir){
      config.publish.remoteDir = path.join(path.dirname(config.publish.remoteDir),dir).split(path.sep).join("/");
      console.log(config.publish.remoteDir);
   }
   //校验输出地址是否合法
   if(!dirPattern.test(config.publish.remoteDir)){
      throw new Error("上传地址非法");
      return
  }
   if(config.publish){
      if(!config.publish.remoteDir){
        throw new Error("请配置远程文件夹地址")
        return
      }

      if(!config.publish.localSourcePath){
          config.publish.localSourcePath = path.join(process.cwd(),"dist");
      }

      if(!config.publish.serverConfig.host||!config.publish.serverConfig.username||!config.publish.serverConfig.password){
        throw new Error("请配置远程服务器信息")
        return
      }

      return config;
   }else{
      return;
   }
};
startUpload(config);

```
