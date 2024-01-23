# Docker Agent with Jenkins Agent in Centos7.5.1804
## Usage
Reference from [Jenkins Document](https://www.jenkins.io/doc/book/using/using-agents/#on-linux):   
docker run -d --rm --name=agent1 -p 22:22 \\   
**-e"JENKINS_AGENT_SSH_PUBKEY=[your-public-key]"** \\   
**\<Built Image Name\>**

## Include packages and libraries
### rpm packages(Latest in yum if not specify version)
 - Subversion(1.14)
 - Python3
 - CMake
 - Make
 - Git
 - GCC and GCC-C++
 - openssh-server

### python3 libraries
 - ply