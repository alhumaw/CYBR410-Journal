---
title: Your Awesome Academic Title
abstract: Lorem markdownum corpus. Date vastarumque artis a incepto Quodsi,
  pressit diversaque tersit excita. Ponti posset quo atro. Ama ungues quo via
  quaerenti culmine haesit moenibus iugum, pluribus flumina ingens.
authors:
  - name: Leonardo V. Castorina
    affiliation: School of Informatics
    institution: University of Edinburgh
    email: justanemail@domain.ext
    address: Edinburgh
  - name: Coauthor
    affiliation: Affiliation
    institution: Institution
    email: coauthor@example.com
    address: Address
reference-section-title: References
---

## Alexander Moomaw - Lab 2 (Containers)

### Overview:  

> Over this week, you have been working on cgroups, namespaces, and chroot.
> In this lab you will create a container and use these tools to limit certain aspects.  

### Getting started:  
```
Get a virtual machine instance of a version of Linux and set it up on Virtual  
Box. Ubuntu is a good choice.  

Create a directory to serve as the root filesystem for the container:  
mkdir rootfs  

Populate the root filesystem with necessary files and directories. You can  
use various tools for this such as buildah, buildroot, debootstrap, YUM/DNF.  
Debootstrap is a good choice for Debian/Ubuntu systems.  
```

- To populate the root filesystem, I used this command:
	- `debootstrap stable rootfs http://deb.debian.org/debian/`
#### Part 1:  
- **Try changing your hostname:**  
	- sudo hostname newhost  
- **What output do you get when you run hostname?**  
- ![[Pasted image 20240410161113.png | left]]
	- **Is this change reflected across terminals?**
		- **When you run it and then open a new terminal?**
			- Running the command hostname, then opening a new terminal displays the hostname as *newhost*
			- ![[Pasted image 20240410161235.png]]
		- **When you open a new terminal and then run it?**  
			- Opening a new terminal, then running it again - It does persist across terminals.
			- ![[Pasted image 20240410161305.png]]
- **Run:**  
	- sudo unshare --uts  
- **Try changing your hostname again:**  
	- hostname abcd  
- **Is this change reflected across terminals? Why or why not?**  
- ![[Pasted image 20240410161439.png]]
	- This change is not reflected across terminals because with unshare we've created our own UTS namespace, so the hostname change is isolated to our rootfs.
- **Use chroot to change the root directory to the container:** 
	- sudo chroot rootfs /bin/bash  
- **Try changing your hostname one more time. Is this change reflected across  terminals? Take a screenshot.**  
	- ![[Pasted image 20240410162149.png]]
	- unless we explicitly create a new namespace for uts for our chrooted container it will be shared with our host system.
	- ![[Screenshot 2024-04-11 at 2.45.38 PM.png]]
- **Use chroot again to get into a new chrooted container and look at all processes.**
	- I took this statement as: "get into a new terminal and try to chroot into rootfs again"
	- ![[Pasted image 20240410163200.png]]
- **Whoops, you’ll need to run mount -t proc proc /proc. Why would you need to do this?**  
	- This error happens because we need to load our kernel and system processes visible from inside our chrooted environment. Doing this allows us to access tools like ps or top within our isolated environment.
- **Try running a program with a user-triggered exit condition (such as top) in  another terminal. Do you see this process running from the chroot container?**  
	- I do see the process from inside the container, this is because we allowed our chrooted environment access to our /proc directory
		- Inside another terminal, run `top`
		- Inside our chroot directory, run `ps -aux`
	- ![[Pasted image 20240410164152.png]]
- **Use unshare to create a new containerized environment that puts the container in its own pid namespace with its own root from rootfs (hint: use --pid).**  
- **Do you still see the running process from within the container? Take a screenshot!**  
	- ![[Pasted image 20240410164800.png]]
	- I still see the running process of top from within the container.
	- I made a new terminal, ran `sudo unshare --pid -f` and the top command was still visible to our container. 
	- This is likely because we are sharing our proc directory with unshare.
	- To get our container in it's own pid namespace, we have to explicitly assign the container it's own proc directory:
		- `sudo unshare --pid --fork --mount-proc=$PWD/rootfs/proc chroot rootfs /bin/bash`
		- ![[Pasted image 20240415105808.png]]
		- By placing the container in it's own pid namespace with it's own proc directory, we are no longer sharing PIDs with our container. The top terminal in the screenshot is our containerized environment, the bottom terminal is our host machine running top.

#### Part 2 - Control groups:  
- **First, create a cgroup directory and mount it with the controllers used to manage CPU resources.**  
	- ![[Pasted image 20240410165316.png]]
- **Create the control group. This can be done directly in the cgroup filesystem or with the cgcreate command.**  
	- `echo "+cpu" > cgroup.subtree_control`
	- `mkdir cpu_c`
- **Now, you want to set a quota to limit that container’s CPU usage (try 50% or even less than that)! You may use cgset for this (or any other method you discovered).**  
	- ![[Pasted image 20240410173621.png]]
	- Just to provide context to this command, the first number is the amount of time our process can run in one period in microseconds. The second number is the length of our period in microseconds. 
	- So, if we set our first number to 500000, that is .5 seconds. If we set our second number to 100000, that is 1 second. So we're essentially saying that any process in this subgroup is allowed to use up 50% of every period that occurs. 
- **Use unshare, chroot, and cgexec together to associate your container with the CPU constraint you assigned, changing the root directory to your populated  container directory. Use the correct unshare flag to isolate the container’s processes.**  
	- ![[Pasted image 20240415110732.png]]
	- The command assigns the container to the cgroup and gives the chrooted directory it's own pids
	- We can verify that the processes were assigned correctly by looking in `/sys/fs/cgroup/cpu_c`
	- ![[Pasted image 20240410175357.png]]

#### Part 3:  
- **Now that you’re in a container with its CPU usage constrained, run a resource-  intensive process of your choosing.**  
	- For this example we'll use `stress` once again. Inside the container, we'll download stress
		- `apt install stress`
	- Once downloaded, all we have to do is start the service, specify x amount of cpu workers
		- `stress -c 4 &`
	- ![[Pasted image 20240410175826.png]]
- **Outside the container, run top again. Is the process listed? What is the CPU  usage? Take a screenshot! What is the PID of the process from inside the container?**  
- ![[Pasted image 20240415111108.png]]
- We can see that the process are indeed getting their cpu limited, indicating that the cgroup assignment was successful. The PIDs of the processes increment from 15764 to 15767. Inside of our chrooted environment the PIDs unique to the environment:
- ![[Pasted image 20240415111154.png]]

#### Write up  
- **Answer the above questions. Additionally answer the following:**  
	- **How does chroot work? How is it related to unshare? Do they work together?**
		- `chroot` works by changing the root directory of the current running process. What this means is the process cannot see or access files outside of its new root directory unless specifically mounted into the new directory
		- `unshare` works by creating individual namespaces for processes that are separate from it's parent. This works in a way such that one set of processes sees one set of resources while another set of processes sees a different set of resources.

	- **Is chroot a secure method of containerization? If yes, why? If not, what could be supplemented to make it a more secure approach?** 
		- `chroot` on it's own is not a secure method of containerization. We have to supplement this service with other tools such as `cgroups` to limit resource utilization, and `namespaces` to further isolate host resources from the container.
	- **Which of the programs you used changes the namespace?**
		- The program used to change the namespace of a process is `unshare`:
			- `sudo unshare -p -f --mount-proc /bin/bash` creates a new namespace with it's own unique process identifiers.

##### Sources Cited:
- https://ericchiang.github.io/post/containers-from-scratch/
- https://www.nginx.com/blog/what-are-namespaces-cgroups-how-do-they-work/
- https://linux.die.net/man/1/stress
- https://wiki.debian.org/Debootstrap
- https://unix.stackexchange.com/questions/588789/changing-the-hostname-in-a-chroot-environment-also-changes-the-hostname-outside
