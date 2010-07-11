---
layout: post
title: Install Java on Ubuntu Lucid Programatically 
---

# {{page.title}}

<span class="meta">June 10 2010</span>

The following commands install Sun's Java 6 JRE on Ubuntu Lucid without the need for the user to respond to interactive prompts. This approach has been tested specifically on the Amazon EC2 AMIs `ami-2d4aa444` and `ami-fd4aa494`.

{% highlight sh %}
sudo add-apt-repository "deb http://archive.canonical.com/ lucid partner"

cat << EOD | sudo debconf-set-selections
sun-java5-jdk shared/accepted-sun-dlj-v1-1 select true
sun-java5-jre shared/accepted-sun-dlj-v1-1 select true
sun-java6-jdk shared/accepted-sun-dlj-v1-1 select true
sun-java6-jre shared/accepted-sun-dlj-v1-1 select true
EOD

sudo dpkg --set-selections << EOS
sun-java6-jdk install
EOS

sudo apt-get update
sudo apt-get install sun-java6-jre -y
sudo update-java-alternatives -s java-6-sun --jre

java -version
{% endhighlight %}

The addition of `parter` repository to the `apt` sources list is needed because the Sun Java JDK is not available in the default `apt` repositories on Lucid. The `debconf-set-selections` and `dpkg --set-selections` calls eliminate the need for user interaction during the Java install. Finally, the `update-java-alternatives` line ensures that the `java` command on the system in question uses the Java 6 Sun JRE.
