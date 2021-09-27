### 自动发布



### 背景
团队进行组件化开发后，会产生大量的组件，每次发版的组件数量都可能有十几甚至几十个，手动发布的话，就是各个业务的owner打自己负责组件的tag，还要分清各个组件的依赖关系，这样费时费力，重复人工劳动。所以需要一个自动化工具来解放这部分人力。业务owner只需要在组件发布失败时，修复组件的问题即可。

### 发布前准备
在自动发布组件之前，我们约定所有待发布组件的代码都要合并到develop分支。

### 发布步骤
在开始打tag之前，我们会根据组件的依赖关系，来整理出待发布组件的优先级，从1000开始递减，优先级高的先打tag。得益与gitlab的分布式架构，我们可以对统一优先级的组件库同时进行发布操作。节约了很多时间。

* 开始要先读取gitlab远端项目的Podfile文件，并生成一个本地文件（生成本地文件读写时，需要转一下utf-8，不然可能会导致格式不正确，写入失败）
```
def file_contents(project_id, file, branch)
  file_contents ||= Gitlab.file_contents(project_id, file, branch)
rescue Gitlab::Error::NotFound => error
  raise error.message
end

def podfile
  contents = file_contents(@project_id, @file, @branch)
  contents.force_encoding('UTF-8')
  temp_local_file = TempLocalFile.new(contents, @file)
  temp_local_file.write
  full_path = temp_local_file.full_path

  @podfile ||= begin
    podfile = Pod::Podfile.from_ruby(full_path, contents)
    podfile
  end
end
```

* 通过Podfile对象读取需要发布的组件，取出来的组件会去远端读取podspec文件，然后生成本地文件，并返回一个本地文件路径数组，后面就是通过这个本地文件来发布组件
```
def tmp_podspecs
  untagged_git_dependencies = podfile.dependencies.select { |dependency| dependency.external? &amp;&amp; dependency.external_source[:tag].nil? &amp;&amp; dependency.external_source[:branch] == Constant.DefaultTagFromBranch}

  spec_files = Parallel.map(untagged_git_dependencies, in_threads: 1) do |dep| 
    file_name = "#{dep.name}.podspec"
    podspec_contents = file_contents(project_name, file_name, branch)
    temp_local_spec_file = TempLocalFile.new(podspec_contents, file_name)
    temp_local_spec_file.write
    path = temp_local_spec_file.full_path
    spec = Pod::Specification.from_file(path)
    remove_tag(auto_add_tag(spec.version.to_s), project_name)
    path
  end

  spec_files
end
```

* 计算优先级，这里是通过深度遍历来计算的，是比较耗时的一个操作，不过因为组件自动化发布是个后台任务，而且组件的量不会达到一个数量级，时间的考量并不高，后续可以优化一下算法（可参考molinillo）
```
def priority_rank(specs)
  temp_spec_map = Hash.new()
  specs.map do |spec| 
    temp_spec_map[spec.name] = spec
  end
  temp_spec_map_copy = Hash.new()

  while !(eqlBetweenHash(temp_spec_map, temp_spec_map_copy)) do
    # 深拷贝
    temp_spec_map_copy = Marshal.load(Marshal.dump(temp_spec_map))
    specs.map do |spec| 
      temp_spec = temp_spec_map[spec.name]
      spec.dependencySpecs.map do |subspec| 
        temp_sub_spec = temp_spec_map[subspec.name]
        if temp_sub_spec &amp;&amp; temp_sub_spec.name != temp_spec.name
          if temp_spec.priority >= temp_sub_spec.priority - 1
            temp_spec.priority = temp_sub_spec.priority - 1
          end
        end
      end
    end
  end
end
```

* 通过上一步生成的优先级，触发第一优先级（1000）开始打tag
```
def create_tag_by_priority()
  result.each do |row|
    if row["tag_status"].eql?(SpecPublishStatus::WAITING) || row["tag_status"].eql?(SpecPublishStatus::CANCELED) || row["tag_status"].eql?(SpecPublishStatus::FAILED)  
      system "gct update tag #{name} --auto-tag --update-ci #{skip_tag} #{pre}"
    end
  end
end
```

* 从刚才生成的本地文件夹下读取当前组件的podspec文件
```
def get_spec_file
  Dir.chdir(FileBase.tmp_path) do
    spec_file = Pathname.glob("#{@name}.podspec").first
    full_path = spec_file.realpath
    raise "在 #{Dir.pwd} 目录下找不到podspec文件".red if spec_file.nil?
  end
end
```

* 更新ci文件，这里会先判断一下远端有没有ci文件，没有则直接创建，有的话再判断是否需要更新，ci文件是在本地写死的，所以每次都要判断一下本地的ci文件跟远端的是否有变化
```
def update_ci_method
  if check_ci
    remote_content = file_contents("#{@group_name}/#{@name}", '.gitlab-ci.yml', @update_version_branch)
    remote_content.force_encoding('UTF-8')
    strip_remote_content = remote_content.gsub(/\s/,'')
    ci_content.force_encoding('UTF-8')
    strip_ci_content = ci_content.gsub(/\s/,'')

    if strip_ci_content.eql?strip_remote_content
      puts "无需更新CI文件".green
    else 
      edit_file('.gitlab-ci.yml', ci_content, 'update ci')
      mr_to_master_if_need
    end
  else 
    create_file('.gitlab-ci.yml', ci_content, 'init ci')
    mr_to_master_if_need
  end
end
```

* 修改完CI之后会创建MR合并到master，这里会判断是否为空mr
```
def mr_to_master
  Gitlab.create_merge_request("#{@group_name}/#{@name}", "mr", { source_branch: "#{@update_version_branch}", target_branch: "#{@update_to_branch}" })
end

def mr_to_master_if_need 
  isEmptyMR = mr_to_master
  if !isEmptyMR
    accept_merge_request
  end
end

def accept_merge_request
  Gitlab.accept_merge_request("#{@project_id}", @id)
end
```

* 通过正则修改版本号，合并到master分支后，创建tag，并推送，因为我们的CI文件是```only tags```，所以推送tag的时候就会触发CI，开始发布组件
```
def update_podspec_version 
  podspec_content = file_contents(@project_id, @file, @update_version_branch)
  version_regex = /.*version.*/
  version_match = version_regex.match(podspec_content).to_s
  tag_regex = /(?<="|')\w.*(?="|')/
  tag = tag_regex.match(version_match)
  @now_tag = tag
  @new_tag = @tag || auto_add_tag(tag.to_s)
  replace_string = version_match.gsub(tag_regex, @new_tag)
  updated_podspec_content = podspec_content.gsub(version_match, replace_string)
  edit_file(@file, updated_podspec_content, "@config 更新版本号：#{@new_tag}")

  mr_to_master
  accept_merge_request
  create_tag
end

def auto_add_tag(tag)
  tag_points = tag.to_s.split('.')
  raise "tag is nil" if tag_points.length <= 0
  index = tag_points.length - 1
  need_add_point = tag_points[index].to_i + 1
  tag_points[index] = need_add_point == 10 ? 0 : need_add_point
  while need_add_point == 10 &amp;&amp; index > 0
      index = index - 1
      need_add_point = tag_points[index].to_i + 1
      tag_points[index] = need_add_point == 10 &amp;&amp; index != 0 ? 0 : need_add_point
  end
  tag_points.join('.')
end
```

注意：以上所有的修改操作都在develop分支进行，再合并到master分支，不建议直接在master分支上修改代码！！！

#### 通过以上几个步骤实现了组件的自动发布，那么如何判断下一个要打的组件呢？


下面是完整的CI文件，在ci文件中，我们会运行一个```gct update next```的命令，由这个命令控制下一个要打的组件
```
stages:
  - push
  - failure

push_pod:
  stage: push
  script: 
    - echo "开始lint 和 push podspec"
    - gct update dependency $CI_PROJECT_NAME "tag_status" "deploying"
    - gct clean lint $CI_PROJECT_NAME
    - gct repo update 
    - gct repo push $CI_PROJECT_NAME
    - gct update dependency $CI_PROJECT_NAME "tag_version" $CI_COMMIT_REF_NAME
    - gct update next
  only:
    - tags
  tags:
    - iOS

on_failure:
  stage: failure
  script:
    - gct robot finish false
    - gct update dependency $CI_PROJECT_NAME "tag_status" "failed"
  when: on_failure
  only: 
    - tags
  tags:
    - iOS

```

* next命令：从数据库中拿出未标记成功的且优先级最高的组件
  * 取出的数据为空，则说明所有组件都发布成功了，这时我们需要更新podfile文件。
  * 取出数据不为空，则判断是否可以发布下一个优先级的组件了
  * 因为该命令写在CI文件中，所以每个组件打完tag后都会执行该命令
```
sql = "select * from tag where priority = (select MAX(priority) from tag where tag_status != '#{SpecPublishStatus::SUCCESS}' #{where_statement}) and project_version = '#{@version}'" 
result = db.query(sql)

if result.count == 0
  system "gct update podfile"
else
  result.each do |row|
    if row["tag_status"].eql?(SpecPublishStatus::WAITING) || row["tag_status"].eql?(SpecPublishStatus::CANCELED)
      status = SpecPublishStatus::PENDING
      name = row["pod_name"]
      skip_tag = ""
      if row["is_tag"].to_i == 1 
        skip_tag = "--skip-tag"
      end
      system "gct update dependency #{name} tag_status #{status}" 
      system "gct update tag #{name} --auto-tag --update-ci #{skip_tag} #{pre}"
    end
  end
end
```

### 更新Podfile
组件全部发布完成之后，我们还要做一件事，就是更新Podfile文件。

为了更新的时候可以用正则，我们在Podfile中写了个方法来承载待发布组件
```
def dev_pod(pod_name, file = pod_name, type = 'remote', branch = 'develop', modular_headers = true)
  case type
    when 'local'
      pod pod_name, :path => "../#{file}", :inhibit_warnings => false, :modular_headers => modular_headers
    when 'remote'
      pod pod_name, :git => "http://gi-dev.ccrgt.com/ios/#{file}.git", :branch => "#{branch}", :modular_headers => modular_headers
    else
  end
end
```
待发布组件的书写格式就变成了这样
```
dev_pod 'AFNetworking'
```
我们通过正则先把待发布组件所在行都删除
```
remove_regex = /dev_pod .*/
podspec_content = podspec_content.gsub!(remove_regex, "")
```
然后在从数据库中取出当前发布的组件和版本更新组件版本，这里会有个问题，如果是新增组件，在Podfile中并没有占位符。解决的方案是自己写个占位符```# Pods for iLife```，然后把所有新增组件都加到这个占位符下。
```
update_regex = /.*#{name}.*/
update_match = update_regex.match(podspec_content).to_s
update_str = "  pod '#{name}', '#{tag}'#{modular_headers}"
if update_match.empty?
  empty_match = "# Pods for iLife"
  podspec_content = podspec_content.gsub!(empty_match, "#{empty_match} \n#{update_str}")  # Pods for iLife
else 
  podspec_content = podspec_content.gsub!(update_match, update_str)
end
```

通过以上步骤完成对Podfile的更新，并推送到远端

### bugfix
在最开始的版本实践中，基本上每次发版都会发现工具不足的地方，也对这些bug做了整理修复，经过十几个小版本的迭代，目前还是比较稳定的执行发布任务的。下面列出修复的几个重点问题：

|                           问题描述                           |                           修复过程                           |  状态   |
| :----------------------------------------------------------: | :----------------------------------------------------------: | :-----: |
|             打tag的库超过五个时，会重复打第六个              |              将CI并发数量增加到50个，原来是5个               |  fixed  |
|                    sharedtravel 不能打tag                    | 该工程根目录下存在gemfile文件，与脚本冲突，在source上指定ruby版本#ruby=2.6.5 source "https://gems.ruby-china.com/" |  fixed  |
|              报错之后没有更新数据库会重复打tag               |                           sql问题                            |  fixed  |
|                     不要每次都update ci                      |           对比本地CI文件和远端CI文件，不同，则更新           |  fixed  |
|                        是否跳过打tag                         | 在数据库中加上标记字段，若已标记打过tag的组件，重新打tag，会删除之前的tag，重新push，使用与打tag失败，修改后重新打tag的组件 |  fixed  |
|                  不开启modular_headers 集合                  | 因为后面做了OC和Swift混编，但并不是所有的库都可以开启混编，这样会涉及到lint参数配置不一样的问题，导致未开启混编的库lint失败 |  fixed  |
| 其他group的库会找不到，比如ios-thirdpart group下的库，默认为ios group | 因为我们的gitlab有两个组，一个是第三方库（ios-thirdpart），一个是二方库（ios），三方库修改的几率不大，不过有时候也会有修改，所以要开启不同的组的CI，设置不同的tags |  fixed  |
|   开始就报错会导致只向数据库写入了project表，没有写入tag表   |                            单元格                            |  fixed  |
|             已经打过的库要重新打tag， reset命令              | 这个优先级不高，而且一般都是违规操作导致的，也就是在打完tag之后还在修改podspec，导致整个优先级发生变化的，暂时未修复 | pending |
|               重新计算优先级 --retry-priority                |                        承接上一个问题                        | pending |

### 总结
gct脚本工具目前属于个人开发维护，所以代码和逻辑都会有很多问题，经过十几个版本的迭代，目前来看还是能比较好的完成自动发布，进行版本迭代的，后续还会继续迭代优化。

### 下一阶段的目标
* 接入webhook，处理更多的异常流程
* 接入zeus，可视化图形页面操作
* 拆分gct工具中的常用工具部分，将gct作为单独的自动发布工具使用
* 使用Ruby的Symbol特性对代码部分改造
* 参考Cocoapods --verbose对终端输出做优化

## 参考链接
* [Cocoapods官网](https://cocoapods.org/)
* [语义化版本2.0.0](https://semver.org/lang/zh-CN/)
* [gems文档](https://www.rubydoc.info/gems/)
* [rubygems文档](https://guides.rubygems.org/)
* [Ruby中文](http://www.ruby-lang.org/zh_cn/documentation/)
* [Ruby Guides](https://www.rubyguides.com/)
* [Ruby China](https://ruby-china.org/)
* [Gitlab文档](https://docs.gitlab.com/ee/ci)