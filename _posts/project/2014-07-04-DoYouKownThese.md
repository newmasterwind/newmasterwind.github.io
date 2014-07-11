---
layout: post
title : 开发小经验
description : 记录下一些开发过程中遇到的小烦恼小问题，以作备忘。
category : project
---


> 这里记录的都是一些比较幼稚的问题，看了您觉得幼稚的话，别当真，见笑了~。~   
> 

###SVN提交问题


- **为啥我用eclipse svn插件checkout的工程不能用SVN客户端提交啊？为啥工程文件夹上没显示svn的标示呢？**  

　**答**：SVN客户端的版本号跟Eclipse svn插件的版本号不一致，可以在工程文件夹上右击-->svn upgrade working copy即可。
<hr/>

###工程代码庞大
- **公司给了我一个系统project，代码好庞大啊！完全无从下手毫无头绪，怎么办？**

　**答**：首先从整体对用到的技术有一定了解，然后从web.xml入手，单点击破，沿着一个功能点走进去，无论后端的一个方法多么长多么复杂，硬着头皮看下去，多看几个方法就会对系统的功能、类代表的意义、有哪些数据库、都是干嘛的 这些有整体了解，以后的开发就非常熟悉了。 **总而言之,耐心和毅力在一开始是最重要的。**
<hr/>
###用户登录状态判断
- **用户登录状态判断只用session判断就可以了么？**

　**答**：答案是否定的，有可能用户并没有关闭浏览器但是服务器端session超时了，这时候如果仅靠session判断登录状态可能失效。此时可以配合用户浏览器端会话cookie（内存中）进行登录状态检查，会话cookie的生成方法是：不给cookie设置失效时间。判断示例代码如下：

	public String getCurrentLoginUserName()
		{
			String userName = null;
			if (session.containsKey(SESSION_LOGIN_USER_KEY))
			{
				//从session中获取
				userName = (String) session.get(SESSION_LOGIN_USER_KEY);
			}
			else
			{
	
				//从会话cookie中获取
				String[] cookSSn = CookieUtil.getSsnFromCookie(request);
				userName = cookSSn[1];
			}
	
			if (userName != null)
			{
				return userName.toLowerCase();
			}
			else
			{
				return userName;
			}
		}
<hr/>

