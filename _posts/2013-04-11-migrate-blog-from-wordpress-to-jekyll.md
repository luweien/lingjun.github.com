---
layout: post
title: "从Wordpress迁移博客到Jekyll"
description: "迁移 jekyll wordpress"
category: "技术"
tags: [迁移, jekyll, wordpress]
---
{% include JB/setup %}

在这里的博客部署好了之后，尝试着将wordpress中的blog也迁移过来了，内容不多，但也费了一番周折。

[Jekyll提供的迁移方案](https://github.com/mojombo/jekyll/wiki/blog-migrations)


### 1. 基本内容迁移

+ 将wordpress文章导出为xml文件；
+ 使用[exitwp](https://github.com/thomasf/exitwp)转换。
    + 需要安装pyyaml和beautifulsoup。具体平台安装请参考README。
    mac OS上的安装方法：

        sudo python -m easy_install pyyaml
        sudo python -m easy_install beautifulsoup
   + 转换后的文章在build里面。

 <!--break-->

### 2. 复杂格式
如果文章中涉及引用、表格或者嵌套的样式，exitwp 转换出来的 Markdown 会惨不忍睹。这个基本只有靠人肉解决。

### 3. 静态资源
在 Jekyll 的 assets 文件夹下建立了 post-images 文件夹，里面再建立以文章 URL 命名的子文件夹。然后把每篇博客的图片复制进去，并修改文章中相应链接。`苦力活`


### 4. Permlink
 Permlink 样式在升级后有改变，你还需要 Migrate DISQUS Thread 。具体操作是以管理员帐号登录 DISQUS ，点击主页上的 "Admin" 然后点击 "Tools" -> "Migrate Threads" 。具体使用方法里面有详细描述。

### 5. 使用了SSL的站点
Jekyll-Bootstrap 里默认的模板使用了 http 而不是 https 来引用 DISQUS 的 JS 。这么做在 Chrome 上会出现评论无法加载的问题。修改方法是打开 _includes/JB/comments-providers/disqus ，将其中的
    dsq.src = 'http://' + disqus_shortname + '.disqus.com/embed.js';
改为                           </br>
    dsq.src = 'https://' + disqus_shortname + '.disqus.com/embed.js';
再次生成，评论应该可以正常显示了。

### 6.高级
如果你是程序员，你就不会手动的来做，而一定是写一个程序来完成这个时间。如果你的文章不多，或者图片更少，也许手动更新更快，但是程序员是又懒又坦言重复劳动的人，

下面的这个Gist, 首先从旧的博文里提取出来图片链接，然后保存到本地，按日期分组，再把旧的链接更新为新的地址。

# encoding: utf-8

	require 'open-uri'
	require 'iconv'

	dir = Dir.getwd.to_s
	puts dir
	entries = Dir.entries(dir + '/_posts')
	ic = Iconv.new('UTF-8//IGNORE', 'UTF-8')
	your_own_domain = "http://fengqijun.me"
	puts "starting download images..."
	images_file_hash = Hash.new
	entries.each do |e|
		file_path = dir + '/_posts/' + e.to_s
		unless File.directory?(file_path)
			file = File.open(file_path)
			contents = ic.iconv(file.read)
			file.close
			contents.scan(Regexp.new(your_own_domain + '(.*?)\.(jpg|jpeg|png)')).each do |src|
				image_url = your_own_domain + "#{src[0]}.#{src[1]}"

				images_file_hash[image_url] = file_path
				comps = image_url.split('/');
				dir_path = dir + '/_posts/' + comps[-3] + '-' + comps[-2]
				if not File.directory?(dir_path)
					Dir.mkdir(dir_path, 0777)
				end
				index = image_url.rindex('/')

				puts "saving to #{dir_path}"
				File.open(dir_path + '/' + comps[-1], "wb") do |saved_file|
					open(image_url, 'rb') do |read_file|
						saved_file.write(read_file.read)
					end
				end
			end
		end
	end

	images_file_hash.each do |image_url, file_path|
		new_image_url = image_url.sub('2012/', '2012-').sub('fengqijun.me/wp-content/uploads', 'images.fengqijun.me')
		puts new_image_url
		file = File.open(file_path, "r")
		contents = ic.iconv(file.read)
		file.close
		puts new_image_url
		contents = contents.gsub(image_url, new_image_url)
		new_file = File.open(file_path, "w+")
		new_file.write(contents);
		new_file.close
	end



> 参考文章
>
[告别WordPress,拥抱Jekyll](https://idndx.com/2013/01/19/migrated-from-wordpress-to-jekyll/)</br>
[告别wordpress，拥抱jekyll---阳志平](http://www.yangzhiping.com/tech/wordpress-to-jekyll.html)
</br>
[从Wordpress迁移到Jekyll](http://fengqijun.me/blog/2012/11/17/move-from-wordpress-to-jekyll)

