---
title: 使用space管理provisionProfile
date: 2019-10-25 15:58:38
tags:
---

#### 痛点

日常iOS开发中，描述文件的管理是一个费时的事情。通常开发者账号仅有部分开发者有权限可以操作，当添加一个测试设备时，需要更新所有的`development`和`ADHoc`描述文件，下载到本地，并替换本地的描述文件，如果有CI系统还需要更新CI上的描述文件。对于区分内外网的公司，可能中间还需要切换网路等操作。总之，这是一件繁琐的事情。

### spaceship

[spaceship](https://github.com/fastlane/fastlane)作为`fastlane`中的一个组件，可以帮助我们处理开发者账号相关的功能：创建appid、更新appid属性、添加设备、更新描述文件等等。



##### Login

```ruby
Spaceship::Portal.login("felix@krausefx.com", "password")

Spaceship::Portal.select_team # call this method to let the user select a team
```

##### App

```ruby
# Fetch all available apps
all_apps = Spaceship::Portal.app.all

# Find a specific app based on the bundle identifier
app = Spaceship::Portal.app.find("com.krausefx.app")

# Show the names of all your apps
Spaceship::Portal.app.all.collect do |app|
  app.name
end

# Create a new app
app = Spaceship::Portal.app.create!(bundle_id: "com.krausefx.app_name", name: "fastlane App")
```

###### 

##### 

##### App Services

```ruby
# Find a specific app based on the bundle identifier
app = Spaceship::Portal.app.find("com.krausefx.app")

# Get detail informations (e.g. see all enabled app services)
app.details

# Enable HealthKit, but make sure HomeKit is disabled
app.update_service(Spaceship::Portal.app_service.health_kit.on)
app.update_service(Spaceship::Portal.app_service.home_kit.off)
app.update_service(Spaceship::Portal.app_service.vpn_configuration.on)
app.update_service(Spaceship::Portal.app_service.passbook.off)
app.update_service(Spaceship::Portal.app_service.cloud_kit.cloud_kit)
```



##### App Groups

```ruby
# Fetch all existing app groups
all_groups = Spaceship::Portal.app_group.all

# Find a specific app group, based on the identifier
group = Spaceship::Portal.app_group.find("group.com.example.application")

# Show the names of all the groups
Spaceship::Portal.app_group.all.collect do |group|
  group.name
end

# Create a new group
group = Spaceship::Portal.app_group.create!(group_id: "group.com.example.another",
                                        name: "Another group")

# Associate an app with this group (overwrites any previous associations)
# Assumes app contains a fetched app, as described above
app = app.associate_groups([group])
```



##### Certificates

```ruby
# Fetch all available certificates (includes signing and push profiles)
certificates = Spaceship::Portal.certificate.all
```

###### Code Signing Certificates

```ruby
# Production identities
prod_certs = Spaceship::Portal.certificate.production.all

# Development identities
dev_certs = Spaceship::Portal.certificate.development.all

# Download a certificate
cert_content = prod_certs.first.download
```

###### Push Certificates

```ruby
# Production push profiles
prod_push_certs = Spaceship::Portal.certificate.production_push.all

# Development push profiles
dev_push_certs = Spaceship::Portal.certificate.development_push.all

# Download a push profile
cert_content = dev_push_certs.first.download

# Creating a push certificate

# Create a new certificate signing request
csr, pkey = Spaceship::Portal.certificate.create_certificate_signing_request

# Use the signing request to create a new push certificate
Spaceship::Portal.certificate.production_push.create!(csr: csr, bundle_id: "com.krausefx.app")
```



###### Create a Certificate

```ruby
# Create a new certificate signing request
csr, pkey = Spaceship::Portal.certificate.create_certificate_signing_request

# Use the signing request to create a new distribution certificate
Spaceship::Portal.certificate.production.create!(csr: csr)
```



##### Provisioning Profiles

###### Receiving profiles

```ruby
##### Finding #####

# Get all available provisioning profiles
profiles = Spaceship::Portal.provisioning_profile.all

# Get all App Store and Ad Hoc profiles
# Both app_store.all and ad_hoc.all return the same
# This is the case since September 2016, since the API has changed
# and there is no fast way to get the type when fetching the profiles
profiles_appstore_adhoc = Spaceship::Portal.provisioning_profile.app_store.all
profiles_appstore_adhoc = Spaceship::Portal.provisioning_profile.ad_hoc.all

# Get all Development profiles
profiles_dev = Spaceship::Portal.provisioning_profile.development.all

# Fetch all profiles for a specific app identifier for the App Store (Array of profiles)
filtered_profiles = Spaceship::Portal.provisioning_profile.app_store.find_by_bundle_id(bundle_id: "com.krausefx.app")

# Check if a provisioning profile is valid
profile.valid?

# Verify that the certificate of the provisioning profile is valid
profile.certificate_valid?

##### Downloading #####

# Download a profile
profile_content = profiles.first.download

# Download a specific profile as file
matching_profiles = Spaceship::Portal.provisioning_profile.app_store.find_by_bundle_id(bundle_id: "com.krausefx.app")
first_profile = matching_profiles.first

File.write("output.mobileprovision", first_profile.download)
```



###### Create a Provisioning Profile

```ruby
# Choose the certificate to use
cert = Spaceship::Portal.certificate.production.all.first

# Create a new provisioning profile with a default name
# The name of the new profile is "com.krausefx.app AppStore"
profile = Spaceship::Portal.provisioning_profile.app_store.create!(bundle_id: "com.krausefx.app",
                                                         certificate: cert)

# AdHoc Profiles will add all devices by default
profile = Spaceship::Portal.provisioning_profile.ad_hoc.create!(bundle_id: "com.krausefx.app",
                                                      certificate: cert,
                                                             name: "Profile Name")

# Store the new profile on the filesystem
File.write("NewProfile.mobileprovision", profile.download)
```



###### Repair all broken provisioning profiles

```ruby
# Select all 'Invalid' or 'Expired' provisioning profiles
broken_profiles = Spaceship::Portal.provisioning_profile.all.find_all do |profile|
  # the below could be replaced with `!profile.valid? || !profile.certificate_valid?`, which takes longer but also verifies the code signing identity
  (profile.status == "Invalid" or profile.status == "Expired")
end

# Iterate over all broken profiles and repair them
broken_profiles.each do |profile|
  profile.repair! # yes, that's all you need to repair a profile
end

# or to do the same thing, just more Ruby like
Spaceship::Portal.provisioning_profile.all.find_all { |p| !p.valid? || !p.certificate_valid? }.map(&:repair!)
```



##### Devices

```ruby
# Get all enabled devices
all_devices = Spaceship::Portal.device.all

# Disable first device
all_devices.first.disable!

# Find disabled device and enable it
Spaceship::Portal.device.find_by_udid("44ee59893cb...", include_disabled: true).enable!

# Get list of all devices, including disabled ones, and filter the result to only include disabled devices use enabled? or disabled? methods
disabled_devices = Spaceship::Portal.device.all(include_disabled: true).select do |device|
  !device.enabled?
end

# or to do the same thing, just more Ruby like with disabled? method
disabled_devices = Spaceship::Portal.device.all(include_disabled: true).select(&:disabled?)

# Register a new device
Spaceship::Portal.device.create!(name: "Private iPhone 6", udid: "5814abb3...")
```



##### Enterprise

```ruby
# Use the InHouse class to get all enterprise certificates
cert = Spaceship::Portal.certificate.in_house.all.first

# Create a new InHouse Enterprise distribution profile
profile = Spaceship::Portal.provisioning_profile.in_house.create!(bundle_id: "com.krausefx.*",
                                                        certificate: cert)

# List all In-House Provisioning Profiles
profiles = Spaceship::Portal.provisioning_profile.in_house.all
```



##### Multiple Spaceships

```ruby
# Launch 2 spaceships
spaceship1 = Spaceship::Launcher.new("felix@krausefx.com", "password")
spaceship2 = Spaceship::Launcher.new("stefan@spaceship.airforce", "password")

# Fetch all registered devices from spaceship1
devices = spaceship1.device.all

# Iterate over the list of available devices
# and register each device from the first account also on the second one
devices.each do |device|
  spaceship2.device.create!(name: device.name, udid: device.udid)
end
```



### 脚本

从上面可以看到，`spaceship`可以帮我们处理几乎所有需要在苹果开发者中心中操作的所有事项，因此完全可以基于`spaceship`编写一个注册新设备并自动更新、下载、同步描述文件的脚本来解决我们的问题。`spaceship`是使用`Ruby`写的，因此我们可以使用`Ruby`来编写这个脚本。



由于脚本可能会分享给内部或外部人员，可以将脚本编写得更通用、安全一些：

- 通用性，其他人员拿到脚本即可使用无需修改

- 安全性，无需将账号、密码等信息硬编码在脚本中

- 便捷性，能较安全地记住密码（比如借助keychain）



由于对`Ruby`相对陌生，可以借助`Dash`查询相应的API，在`Dash`中下载相应的文档即可。

`Ruby`的断点调试可以使用`pry`:

```ruby
gem install pry
```

安装完毕后，在需要断点的地方使用:

```ruby
require 'pry'

name = 'jack'
binding.pry
name += ' jones'
puts "name:${name}"
```

代码运行后，会在设置了binding的地方暂停，即可进行调试。



最终[示例代码](https://github.com/sleepEarlier/spaceshipUpdateProfile):

```ruby
require "spaceship"
# require 'io/console'
require 'Open3'
require 'fileutils'

puts "Enter Your Account:"
account = gets.chomp

# get password from keychain
service = account + "_DeveloperService"
cmd = "security find-generic-password -a $USER -s #{service} -w"
# puts cmd
$pwd = ''
# puts $pwd
Open3.popen3(cmd) do |stdin, stdout, stderr, wait_thr|
    while line = stdout.gets
        line = line.strip
        if line && line.length > 0
            # puts "line: #{line}"
            $pwd = line
        end
    end
end

if not $pwd
    $pwd = ''
end

if $pwd.length > 0
    puts "Use Keychain Password?(y/n)"
    use = gets.chomp
    if use.downcase != 'y' and use.downcase != 'yes'
        $pwd = ''
    end
end

if $pwd.length == 0
    puts "Enter Your Password:"
    $pwd = STDIN.noecho(&:gets).chomp
end

Spaceship.login(account, $pwd)

# save password to keychain
# puts "Updating Keychain"
cmd = "security add-generic-password -U -a $USER -s #{service} -w #$pwd"
# puts cmd
system(cmd)

# 更新设备
fileDir = File.dirname(__FILE__)
deviceFile = File.join(fileDir, "multiple-device-upload-ios.txt")
file = File.open(deviceFile) #文本文件里录入的udid和设备名用tab分隔
puts "\n-ADDING DEVICES"
file.each do |line|
    # puts "line:#{line}"
    arr = line.strip.split(" ")
    # puts "arr=#{arr}"
    udid = arr[0]
    name = arr[1]
    puts "\t-DeviceName:#{name}, udid:#{udid}"
    device = Spaceship.device.create!(name: arr[1], udid: arr[0])
    puts "\t-add device: #{device.name} #{device.udid} #{device.model}"
end

devices = Spaceship.device.all

profiles = Array.new
profiles += Spaceship.provisioning_profile.development.all 
profiles += Spaceship.provisioning_profile.ad_hoc.all

puts "\n-UPDATING PROFILES"
profiles.each do |p|
    puts "\t-Updating #{p.name}"
    p.devices = devices
    p.update!
end


downloadProfiles = Array.new
downloadProfiles += Spaceship.provisioning_profile.development.all 
downloadProfiles += Spaceship.provisioning_profile.ad_hoc.all

puts "\n-DOWNLOADING PROFILES"
downloadProfiles.each do |p|
    puts "\t-Downloading #{p.name}"
    fileName = p.name
    # save to Downloads floder
    downloadPath = File.expand_path("~/Downloads/#{fileName}.mobileprovision")
    File.write(downloadPath, p.download)
    puts "\t-File at: #{downloadPath}"
    # rename and copy to Provisioning Profiles floder
    dest = File.expand_path("~/Library/MobileDevice/Provisioning Profiles/#{p.uuid}.mobileprovision")
    FileUtils.copy(downloadPath, dest)
    puts "\t-Replace #{p.name} in Provisioning Profiles"
end
```




