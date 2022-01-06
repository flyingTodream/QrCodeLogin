## **实现简易的二维码登录效果**

#### **原理**

网上关于原理的叙述已经很多了，在这里就不过多叙述，这里主要以实战为主

#### 实现思路

------

PC端发送一个二维码的请求，服务器生成一个UUID，加上自己的密钥，在md5一下（这里MD5没有必要，可以用公司自己的密钥算法生成一个唯一的密钥），然后把生成的密钥当作一个key值存在redis里面，value值就存code的一些信息，并设置过期时间（我这里为了方便没有设置），服务器把生成的code返回给前端。

PC端根据服务器返回的密钥，和确认登陆页面的url生成一个二维码进行展示。

手机端扫码二维码，跳转到相应的页面，拿到PC端传过来的code码，并发送更新二维码扫描状态的请求，当用户点击确认登录时，再次发送更新二维码状态的请求，并把token(登录时生成的)，一起发送至服务器，服务器拿到token，获得用户信息，再把用户信息返回给PC端，至此就完成了二维码登录的效果

#### 实现代码

*我这里为了方便，没有设置redis过期时间，相应前端也没判断二维码是否过期，登录时也没用到token，扫描后可以自己输入用户信息，确认登陆后，你输入的用户信息会反显到PC端的登录页面*

下面请看代码：

------

前端使用了vue框架，首先发送生成二维码接口，然后再轮询二维码状态，

页面代码如下：

```vue
<template>
  <div>
    <div ref="qrcode" id="qrcode" class="code">
      <div class="text">
        <i v-if="isScan" class="el-icon-check"></i>
        {{text}}
      </div>
    </div>
    <div class="text">登陆用户是：{{userName}}</div>
  </div>
</template>
<script>
import QRCode from 'easyqrcodejs'
import axios from 'axios'
export default {
  data() {
    return {
      text: '请扫描二维码',
      userName: '',
      isScan: false
    }
  },
  async mounted() {
    let code = ''
    await axios.get('http://192.168.0.103:8888/qrcode/getQr').then(res => {
      code = res.data.data.code
      const options = {
        //此处是前端确认登页面的url
        text: 'http://192.168.0.103:8080/confim?code=' + code,
        colorDark: '#000',
        colorLight: '#FFF',
        width: 150,
        height: 150,
        autoColor: true
      }
      new QRCode(this.$refs.qrcode, options)
    })
    //生成二维码后，轮询二维码状态（此处用了ES6的async函数）
    const interval = setInterval(() => {
      //此处换成你服务器自己的地址
      axios
        .get('http://192.168.0.103:8888/qrcode/getStatus?code=' + code)
        .then(res => {
          if (res.data.data.status == 'yes') {
            this.text = '扫描成功，请确认登陆'
            this.isScan = true
          }
          if (res.data.data.status == 'success') {
            this.text = '登陆成功'
            this.userName = res.data.data.userName
            clearInterval(interval) //停止
          }
        })
    }, 1000)
  }
}
</script>
<style lang="scss" scoped>
#qrcode {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  margin-top: 100px;
}
.text {
  width: 100%;
  text-align: center;
  padding: 20px 0;
}
.el-icon-check {
  color: #12e318;
  font-size: 18px;
  font-weight: bolder;
}
</style>
```

后台使用的Spring Boot + Mybitis框架

后台代码如下：

```java
	/**
	 * 生成二维码
	 * 
	 * @return
	 */
	@RequestMapping(value = "/getQr", method = RequestMethod.GET)
	public Result getQr() {
		Jedis jedis = new Jedis("localhost");
		System.out.println("连接成功");
		// 查看服务是否运行
		System.out.println("服务正在运行: " + jedis.ping());

		String code = Md5.string2MD5(StringUtils.UUID() + "csii");
		JSONObject data = new JSONObject();
		data.put("status", "no");
		//设置过期时间
		jedis.set(code, data.toString());
		JSONObject json = new JSONObject();
		json.put("code", code);
		return ResultUtils.wrapSuccess(null, json);
	}
	/**
	 * 更新二维码状态
	 * 
	 * @param code  生成的code
	 * @param status 前端传过来的状态
	 * @param userName 用户信息
	 * @return
	 */
	@RequestMapping(value = "/UpdateStatus", method = RequestMethod.GET)
	public Result UpdateStatus(String code, String status, String userName) {
		Jedis jedis = new Jedis("localhost");
		System.out.println("连接成功");
		JSONObject data = new JSONObject();
		String res = jedis.get(code);
		
		//判断code是否存在
		if (jedis.exists(code)) {
			data.put("status", status);
			//如果点击了确认登陆
			if (status.equals("success")) {
				data.put("userName", userName);
			}
			jedis.set(code, data.toString());
		}else {
			//设置过期标记
			data.put("errorCode", 0);
		}
		return ResultUtils.wrapSuccess(null, null);
	}

	/**
	 * 获取二维码状态
	 * 
	 * @param code
	 * @return
	 */
	@RequestMapping(value = "/getStatus", method = RequestMethod.GET)
	public Result getStatus(String code) {
		Jedis jedis = new Jedis("localhost");
		System.out.println("连接成功");
		String data = jedis.get(code);
		JSONObject res = (JSONObject) JSON.parse(data);
		return ResultUtils.wrapSuccess(null, res);
	}
```

前端确认登陆页面代码如下：

```vue
<template>
  <div>
    <el-input v-model="userName" placeholder="请输入用户名"></el-input>
    <el-button @click="login">确认登陆</el-button>
  </div>
</template>
<script>
import axios from 'axios'
export default {
  data() {
    return {
      userName: ''
    }
  },
  mounted() {
    //进入页面即更新二维码状态为扫码成功
    const code = this.$route.query.code
    //此处换成你自己服务器的url
    const url =
      'http://192.168.0.103:8888/qrcode/UpdateStatus?code=' +
      code +
      '&status=yes'
    axios.get(url).then(res => {
      console.log(res)
    })
  },
  methods: {
      
    login() {
      const code = this.$route.query.code
      //此处换成你自己服务器的url
      const url =
        'http://192.168.0.103:8888/qrcode/UpdateStatus?code=' +
        code +
        '&status=success' +
        this.userName
      //更新状态为登录成功
      axios.get(url).then(res => {
        console.log(res)
      })
    }
  }
}
</script>
```

此篇实现了简易的二维码登录过程，代码有很多写的不足的地方，望包容，代码仅供参考，如果有什么需要改进的地方请给我留言，

喜欢就请给我一个start吧
