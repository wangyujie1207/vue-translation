<template>
	<div id="app" class="page-wrap">
		<div class="title-line">
			<h3>i18n批量翻译工具</h3>
			<el-button type="warning" size="mini" icon="el-icon-s-tools" @click="dialogFormVisible = true">配置</el-button>
		</div>
		<el-row class="top" :gutter="10">
			<el-col :span="12"><textarea v-model="old_text" @input="onInput()" placeholder="请输入要翻译的JSON数据，中文" class="input-textarea"></textarea></el-col>
			<el-col :span="12">
				<pre class="to-json" v-html="old_json_html"></pre>
			</el-col>
		</el-row>
		<el-row class="btn-line" :gutter="20">
			<el-col :span="3">
				<el-button type="primary" @click="fanyi('en')">翻译英文</el-button>
			</el-col>
			<el-col :span="3">
				<el-button type="primary" @click="fanyi('kor')">翻译韩语</el-button>
			</el-col>
			<el-col :span="3">
				<el-button type="primary" @click="fanyi('jp')">翻译日语</el-button>
			</el-col>
			<el-col :span="3">
				<el-button type="primary" @click="fanyi('th')">翻译泰语</el-button>
			</el-col>
      <el-col :span="3">
        <el-button type="primary" @click="fanyi('cht')">中文繁体</el-button>
      </el-col>
		</el-row>
		<el-row class="new-json-view">
			<el-col>
				<pre class="to-json" v-html="new_json"></pre>
			</el-col>
		</el-row>
		<el-row class="btn-line" :gutter="20">
			<el-col :span="3">
				<el-button type="success" data-clipboard-target="#bar" class="btn01">复制</el-button>
			</el-col>
		</el-row>
		<h3 class="err-text" v-if="err_arr.length">以下文字未翻译<span>(如果所有单词都未翻译，请检查appid或key配置是否正确，或者查看F12 -> Network定位错误原因)</span></h3>
		<el-row class="new-json-view" v-if="err_arr.length">
			<el-col>
				<pre class="to-json" v-html="err_arr"></pre>
			</el-col>
		</el-row>


		<!-- 配置appid和key -->
		<el-dialog title="配置" :visible.sync="dialogFormVisible">
			<el-form :model="form" :rules="rules" ref="form">
				<el-form-item label="appid" label-width="80px" prop="appid">
					<el-input v-model="form.appid" autocomplete="off"></el-input>
				</el-form-item>
				<el-form-item label="key" label-width="80px" prop="key">
					<el-input v-model="form.key" autocomplete="off"></el-input>
				</el-form-item>
			</el-form>
			<div class="tips">
				<p>1、注册百度翻译开放平台账号 <a href="https://api.fanyi.baidu.com/" target="_blank">https://api.fanyi.baidu.com/</a>,并进行实名认证</p>
				<p>2、开通通用翻译API服务<a href="https://fanyi-api.baidu.com/choose" target="_blank">开通链接</a></p>
				<p>3、在<a href="https://api.fanyi.baidu.com/manage/developer" target="_blank">开发者中心</a>,设置通用翻译服务信息的服务器地址，填写成<a href="https://ip.cn/" target="_blank">本机网络IP</a></p>
			</div>
			<div slot="footer" class="dialog-footer">
				<el-button @click="dialogFormVisible = false">取 消</el-button>
				<el-button type="primary" @click="onConfig()">确 定</el-button>
			</div>
		</el-dialog>
	</div>
</template>

<script>
	import md5 from 'md5'
	import Clipboard from 'clipboard' //一键复制插件
	let str_arr = [];
	const interval = 120; //每120毫秒请求一次
	export default {
		name: 'app',
		data() {
			return {
				dialogFormVisible: false, //配置弹窗
				form: {
					appid: "20191015000341768", //百度翻译的APP ID
					key: "XsYxjZiVq52YQkBcWPXK", //百度翻译的开发者秘钥
				},
				rules: {
					appid: [
						{ required: true, message: '请输入appid', trigger: 'blur' }
					],
					key: [
						{ required: true, message: '请输入key', trigger: 'blur' }
					],
				},
				old_text: "", //输入框输入的json字符串
				old_json: "", //输入的json字符串转成的json数据
				old_json_html: "", //右边格式化显示的json数据
				new_json: "", //翻译之后的json数据
				err_arr: [], //翻译错误的数组
			}
		},
		mounted() {
			this.ToClipboard(); //复制
			if (!this.form.appid || !this.form.key) {
				this.dialogFormVisible = true;
			}
		},
		methods: {
			//当输入框输入时
			onInput() {

				// 如果输入为空或
				if (this.old_text == "") {
					this.old_json = ""
					this.old_json_html = ''
					return false
				}
				if (isJSON(this.old_text)) {
					this.old_json = JSON.parse(this.old_text) //把输入的json字符串转成json数据
					this.old_json_html = syntaxHighlight(this.old_json) //右边格式化显示的json数据（html格式）
				} else {
					this.old_json = "输入的不是JSON数据"
					this.old_json_html = "输入的不是JSON数据"
				}
			},
			//提交翻译
			fanyi(to) {
				let vm = this;
				if (!this.form.appid || !this.form.key) {
					this.$message.error("请配置appid及key参数")
					return false;
				}
				if (this.old_json == '' || JSON.stringify(this.old_json) == "{}") {
					this.$message.error("请输入待翻译的JSON")
					return false
				}
				this.new_json = deepClone(this.old_json) //深度克隆新的json 翻译回来的结果在这个json更改
				str_arr = [];
				this.err_arr = []; //清空错误的数组
				let arr = parseJson(vm.new_json)
				arr.forEach(function(e, index) {
					setTimeout(function() {
						var tmp = Date.parse(new Date()).toString();
						tmp = tmp.substr(0, 10); //10位随机数
						let k = eval(`vm.new_json${e}`);
						let params = {
							q: k, //要翻译的字符串（词）
							from: "zh",
							to: to,
							appid: vm.form.appid,
							salt: tmp,
							sign: "",
						}
						let sign = vm.form.appid + params.q + params.salt + vm.form.key;
						sign = md5(sign)
						params.sign = sign
						let api_url = "/api/api/trans/vip/translate?q=" + params.q + "&from=" + params.from + "&to=" + params.to + "&appid=" + vm.form.appid + "&salt=" + params.salt + "&sign=" + params.sign
						vm.$http.get(api_url)
							.then(res => {
								if (res.data.trans_result[0]['dst']) {
									eval(`vm.new_json${e} = "${res.data.trans_result[0]['dst']}"`)
								} else {
									//未翻译的
									vm.err_arr = [...vm.err_arr, ...[k]]
								}
							})
							.catch(err => {
								vm.err_arr = [...vm.err_arr, ...[k]]
							})
					}, index * interval);
				})
			},
			//复制
			ToClipboard() {
				let vm = this
				// 多个dom元素复制
				var btn = document.getElementsByClassName('btn01');
				var clipboard = new Clipboard(btn);
				clipboard.on('success', function(e) {
					vm.$message({
						message: '复制成功',
						type: 'success'
					});
				});
				clipboard.on('error', function(e) {
					vm.$message({
						message: '复制失败',
						type: 'warning'
					});
				});
			},
			//提交配置
			onConfig() {
				this.$refs.form.validate((valid) => {
					if (valid) {
						this.dialogFormVisible = false;
					} else {
						console.log('error submit!!');
						return false;
					}
				});
			}
		}
	}


	function syntaxHighlight(json) {
		if (typeof json != "string") {
			json = JSON.stringify(json, undefined, 2)
		}
		json = json.replace(/&/g, "&amp;").replace(/</g, "&lt;").replace(/>/g, "&gt;")
		return json.replace(/("(\\u[a-zA-Z0-9]{4}|\\[^u]|[^\\"])*"(\s*:)?|\b(true|false|null)\b|-?\d+(?:\.\d*)?(?:[eE][+\-]?\d+)?)/g, function(match) {

			var cls = "number"
			if (/^"/.test(match)) {
				if (/:$/.test(match)) {
					cls = "key"
				} else {
					cls = "string"
				}
			} else if (/true|false/.test(match)) {
				cls = "boolean"
			} else if (/null/.test(match)) {
				cls = "null"
			}
			return '<span class="' + cls + '">' + match + "</span>"
		})
	}

	function isJSON(str) {
		if (typeof str == "string") {
			try {
				var obj = JSON.parse(str)
				if (typeof obj == "object" && obj) {
					return true
				} else {
					return false
				}
			} catch (e) {
				return false
			}
		}
	}

	function parseJson(jsonObj, s = null) {
		let i = []
		// 循环所有键
		for (var v in jsonObj) {
			// 1.判断是不是字符串
			if (typeof jsonObj[v] == "object") {
				//如果是对象重新调用parseJson
				let s2 = "";
				if (s) {
					s2 = `${s}['${v}']`
				} else {
					s2 = `['${v}']`
				}
				parseJson(jsonObj[v], s2)
			} else {
				if (s) {
					str_arr.push(`${s}['${v}']`)
				} else {
					str_arr.push(`['${v}']`)
				}

			}
		}
		return str_arr
	}

	function addToast(text, status = 'success') {
		$.Toast("提示", text, status, {
			stack: true,
			has_icon: false,
			has_close_btn: true,
			fullscreen: false,
			timeout: 3000,
			sticky: false,
			has_progress: true,
			rtl: false,
		});
	}

	//深拷贝
	function deepClone(target) {
		// 定义一个变量
		let result;
		// 如果当前需要深拷贝的是一个对象的话
		if (typeof target === 'object') {
			// 如果是一个数组的话
			if (Array.isArray(target)) {
				result = []; // 将result赋值为一个数组，并且执行遍历
				for (let i in target) {
					// 递归克隆数组中的每一项
					result.push(deepClone(target[i]))
				}
				// 判断如果当前的值是null的话；直接赋值为null
			} else if (target === null) {
				result = null;
				// 判断如果当前的值是一个RegExp对象的话，直接赋值
			} else if (target.constructor === RegExp) {
				result = target;
			} else {
				// 否则是普通对象，直接for in循环，递归赋值对象的所有值
				result = {};
				for (let i in target) {
					result[i] = deepClone(target[i]);
				}
			}
			// 如果不是对象的话，就是基本数据类型，那么直接赋值
		} else {
			result = target;
		}
		// 返回最终结果
		return result;
	}
</script>

<style>
	* {
		margin: 0;
		padding: 0;
		box-sizing: border-box;
	}

	body {
		padding: 20px;
	}

	.page-wrap {
		max-width: 1200px;
		margin: 0 auto;
	}

	.title-line {
		display: flex;
		align-items: center;
		justify-content: space-between;
	}

	.top {
		margin-top: 20px;
	}

	.input-textarea {
		width: 100%;
		height: 200px;
		resize: none;
		padding: 10px;
		border-radius: 4px;
		overflow: scroll;
		white-space: nowrap;
		outline: none;
		border: 1px solid #f0f0f0;
	}

	.to-json {
		width: 100%;
		height: 200px;
		overflow: scroll;
		background-color: #f0f0f0;
		border-radius: 4px;
		padding: 20px;
	}

	.to-json .string {
		color: green;
	}

	.to-json .number {
		color: darkorange;
	}

	.to-json .boolean {
		color: blue;
	}

	.to-json .null {
		color: magenta;
	}

	.to-json .key {
		color: red;
	}

	.new-json-view {
		margin-top: 20px;
	}

	.new-json-view .to-json {
		margin: 0;
		width: 100%;
		height: 200px;
		background-color: #f0f0f0;

	}

	.btn-line {
		margin-top: 12px;
	}

	button {
		display: block;
		line-height: 40px;
		padding: 0 30px;
		border: none;
	}

	*::-webkit-scrollbar {
		/*滚动条整体样式*/
		width: 6px;
		/*高宽分别对应横竖滚动条的尺寸*/
		height: 0px;
	}

	*::-webkit-scrollbar-thumb {
		/*滚动条里面小方块*/
		border-radius: 0px;
		/* -webkit-box-shadow: inset 0 0 5px rgba(0,0,0,0.2); */
		background: #818181;
	}

	*::-webkit-scrollbar-track {
		/*滚动条里面轨道*/

		/* -webkit-box-shadow: inset 0 0 5px rgba(0,0,0,0.2); */

		border-radius: 0px;

		background: #ffffff;

	}

	.err-text {
		margin-top: 20px;
		color: #f11010;
	}

	.err-text span {
		font-size: 12px;
		color: #666666;
		font-weight: normal;
		padding-left: 10px;
	}

	.tips {
		display: flex;
		flex-direction: column;

	}

	.tips p {
		display: flex;
		align-items: center;
		padding: 2px 0;
		font-size: 12px;
	}

	.tips p a {
		color: #409EFF;
		text-decoration: none;
		padding: 0 2px;
	}
</style>
