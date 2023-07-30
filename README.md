# DifferentialAnalysis

This is the repository that corresponds to the **Differential File System Analysis for the Quick Win** Talk to be presented at [SANS DFIR Summit 2023](https://www.sans.org/cyber-security-training-events/digital-forensics-summit-2023/) on August 4, 2023.  A copy of the slides are available [here](SlidesPlaceholder.html).  

## Abstract

Mature DevOps organizations use continuous integration/continuous delivery (CI/CD) techniques to deliver a hardened virtual machine “gold image” to production that does not need any additional configuration on first boot and is ready to join the cluster of virtual machines in the backend pool of its designated load balancer. This approach offers several significant security advantages, but it can also speed up the time to do a forensic analysis when Differential File System Analysis is employed.

Differential File System Analysis is a technique wherein the storage volume(s) of a VM launched from a gold image are mounted read-only to a forensic workstation and are used as a basis for comparison against the forensic copies of the storage volume(s) of a VM that is suspected to be compromised. A reference hash set of all files on the gold image can be prepared in advance by the CI/CD pipeline and stored until needed. Any hashes on the compromised system that are not found in the reference hash set are either new or altered.

Although this talk will demonstrate how to use the Differential File System Analysis technique and open-source software to investigate a compromised AWS EC2 instance, this technique is effective on any system launched recently from a gold image. The talk concludes with examples of how the high-level forensic processing steps can be automated to further reduce the time from compromise to analysis.

## Demo Instructions

1. Create a new role called "EC2_Responder" that has the `AmazonEC2FullAccess` and `AmazonS3FullAccess` policies attached.
2. Use the web console to launch an Amazon Linux t2.micro EC2 Instance into the `us-east-1a` availability zone. Name it `PROD_Host`. Create a security group that allows SSH access from just your IP address. 
3. Similarly, use the web console to launch an Ubuntu t2.micro EC2 Instance into the `us-east-1a` availability zone. Attach the EC2_Responder Role to the Instance and Name it `DFIR_Host`. Use the same security group that was created in the previous step.
4. Determine the Volume that is being used by the `PROD_Host` and label it with the `name` tag value set to `PROD_volume`
5. Label the remaining volume with the `name` tag value set to `DFIR_volume`
6. SSH into the PROD_Host and paste the following command:

```
echo "Here is an important file" > /home/ec2-user/important.txt
```

Then exit the SSH session.
7. SSH into the `DFIR_Host`
8. [In the DFIR_Host SSH Session] Update the `DFIR_Host` and install sleuthkit and the AWS CLI and sleuthkit.

```
sudo apt upgrade && sudo apt update -y
sudo apt install -y unzip sleuthkit
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

9. [In the DFIR_Host SSH Session] Set the VOLUME_ID of the Volume that has the name set to `PROD_volume` using the command:

```
PROD_VOLUME=$(aws ec2 describe-volumes --filters "Name=status,Values=in-use" "Name=tag:Name,Values=PROD_volume" --query "Volumes[*].VolumeId" --output text)
echo $PROD_VOLUME
```

> TIP: Drop any of these commands into ChatGPT or Bard for a detailed explaination.

10. [In the DFIR_Host SSH Session] Make a snapshot of the `PROD_volume` and set the `name` tag value to `REFERENCE`using the command as follows:
```
REFERENCE_SNAPSHOT=$(aws ec2 create-snapshot --volume-id $PROD_VOLUME --description "Snapshot of REFERENCE volume created on 2023-07-25 14:07:14 PST" --tag-specifications "ResourceType=snapshot,Tags=[{Key=Name,Value=REFERENCE}]" --query SnapshotId --output text)
echo "wait for it..."
aws ec2 wait snapshot-completed --snapshot-ids $REFERENCE_SNAPSHOT
echo $REFERENCE_SNAPSHOT
```

11. SSH into the `PROD_Host` and run the following infection script:

```
wget https://s3.amazonaws.com/forensicate.cloud-data/dont_peek2.sh
sudo bash dont_peek2.sh 
```

**Note:** The instance will shut itself down in 5 minutes so just let it run. Wait until the instance has shut down before proceeding.

12. Make a second snapshot of the `PROD_volume` and set the `name` tag value to `EVIDENCE` using the command on the as follows:

```
EVIDENCE_SNAPSHOT=$(aws ec2 create-snapshot --volume-id $PROD_VOLUME --description "Snapshot of EVIDENCE volume created on 2023-07-25 14:07:14 PST" --tag-specifications "ResourceType=snapshot,Tags=[{Key=Name,Value=EVIDENCE}]" --query SnapshotId --output text)
echo "wait for it..."
aws ec2 wait snapshot-completed --snapshot-ids $EVIDENCE_SNAPSHOT
echo $EVIDENCE_SNAPSHOT
```

13. Make a `REFERENCE` volume from the `REFERENCE` snapshot.

```
REFERENCE_VOLUME=$(aws ec2 create-volume --snapshot-id $REFERENCE_SNAPSHOT --availability-zone us-east-1a --tag-specifications "ResourceType=volume,Tags=[{Key=Name,Value=REFERENCE}]" --query 'VolumeId' --output text)
echo $REFERENCE_VOLUME
```

14. Similarly, make an `EVIDENCE` volume from the `EVIDENCE` snapshot.

```
EVIDENCE_VOLUME=$(aws ec2 create-volume --snapshot-id $EVIDENCE_SNAPSHOT --availability-zone us-east-1a --tag-specifications "ResourceType=volume,Tags=[{Key=Name,Value=EVIDENCE}]" --query 'VolumeId' --output text)
echo $EVIDENCE_VOLUME
```

15. Make an empty 100 GB volume named `DATA`

```
DATA_VOLUME=$(aws ec2 create-volume --size 100 --availability-zone us-east-1a --volume-type gp2 --tag-specifications "ResourceType=volume,Tags=[{Key=Name,Value=DATA}]" --query 'VolumeId' --output text)
echo $DATA_VOLUME
```

16. Query the Virtual Machine's Instance Metadata service (IMDS) to set the INSTANCE_ID

```
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600") && INSTANCE_ID=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id)
echo $INSTANCE_ID
```

17. Next, Attach the `REFERENCE` volume to `/dev/xvdb`

```
aws ec2 attach-volume --volume-id $REFERENCE_VOLUME  --instance-id $INSTANCE_ID --device /dev/xvdb
```

This should return a result that looks something like:

```
{
    "AttachTime": "2023-07-25T22:08:45.711000+00:00",
    "Device": "/dev/xvdb",
    "InstanceId": "i-01647e9e5121e6552",
    "State": "attaching",
    "VolumeId": "vol-0b97fa76e15e3babe"
}
```

18. Similarly, attach the `EVIDENCE` volume to `/dev/xvdc`

```
aws ec2 attach-volume --volume-id $EVIDENCE_VOLUME  --instance-id $INSTANCE_ID --device /dev/xvdc
```

19. Next, attach the `DATA` volume to `/dev/xvdd`

```
aws ec2 attach-volume --volume-id $DATA_VOLUME  --instance-id $INSTANCE_ID --device /dev/xvdd
```

20. **Switch to root** and inspect the filesystem types using `lsblk -f` 

```
ubuntu@ip-172-31-89-235:~$ sudo su
root@ip-172-31-89-235:/home/ubuntu# lsblk -f
NAME      FSTYPE FSVER LABEL           UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
loop0                                                                             0   100% /snap/amazon-ssm-agent/6312
loop1                                                                             0   100% /snap/core18/2745
loop2                                                                             0   100% /snap/core20/1879
loop3                                                                             0   100% /snap/lxd/24322
loop4                                                                             0   100% /snap/snapd/19122
xvda
├─xvda1   ext4   1.0   cloudimg-rootfs 4513eb34-58e6-408e-8ed7-3d487fe6b35b    5.3G    30% /
├─xvda14
└─xvda15  vfat   FAT32 UEFI            6192-5E23                              98.3M     6% /boot/efi
xvdb
├─xvdb1   xfs          /               3325c0ba-3d91-4d25-bb13-bdc5c47a979a
├─xvdb127
└─xvdb128 vfat   FAT16                 CE70-438F
xvdc
├─xvdc1   xfs          /               3325c0ba-3d91-4d25-bb13-bdc5c47a979a
├─xvdc127
└─xvdc128 vfat   FAT16                 CE70-438F
xvdd
root@ip-172-31-89-235:/home/ubuntu#
```

Note that the UUID of the REFERENCE (xvdb) volume's partitions are the same as the EVIDENCE volume's partitions.

21. Format the new `DATA` volume

```
mkfs -t xfs /dev/xvdd
```

22. Make some mount points for the three volumes

```
mkdir /mnt/reference
mkdir /mnt/evidence
mkdir /mnt/data
```

23. Change the UUID of the `EVIDENCE` volume so that it does not conflict with the UUID of the `EVIDENCE` volume. Without this step, only one of the two volumes will be able to be mounted. 

```
xfs_admin -U $(uuidgen) /dev/xvdc1
```

24. Mount the `REFERENCE` and `EVIDENCE` volumes as read-only and mount the `DATA` as read-write.

```
mount -o ro -t xfs /dev/xvdb1 /mnt/reference
mount -o ro -t xfs /dev/xvdc1 /mnt/evidence
mount /dev/xvdd /mnt/data
lsblk
```

25. Verify that the `REFERENCE` and `EVIDENCE` volumes are read-only and the `DATA` volume is read-write.

```
touch /mnt/reference/tmp/test
touch /mnt/evidence/tmp/test
touch /mnt/data/test
ls /mnt/data/test
rm /mnt/data/test
```

**NOTE:** Shout out to Brian Carrier of Basis Technology for `sleuthkit` and the following references:
* https://www.sleuthkit.org/informer/sleuthkit-informer-6.html#hashes
* https://www.sleuthkit.org/informer/sleuthkit-informer-7.html 

26. Generate the List of MD5 hashes for files on the `REFERENCE` and `EVIDENCE` volumes 

```
mkdir /mnt/data/hashdata && cd /mnt/data/hashdata

# Create REFERENCE Hash Set
find /mnt/reference -type f -print0 | xargs -0 md5sum | tee reference_files.md5

# Create EVIDENCE Hash Set
find /mnt/evidence -type f -print0 | xargs -0 md5sum | tee evidence_files.md5

wc -l *
```

NOTE: May need to fix a stray character as follows:

```
echo; echo "Highlight the stray character in reference_files.md5:"
grep -C3 22369d5c587517e7ff963c164b878f55  reference_files.md5
sed -i 's|^\\||g' reference_files.md5
echo; echo "Show the stray character is fixed in reference_files.md5:"
grep -C3 22369d5c587517e7ff963c164b878f55  reference_files.md5
echo; echo "Highlight the stray character in evidence_files.md5:"
grep -C3 22369d5c587517e7ff963c164b878f55  evidence_files.md5
sed -i 's|^\\||g' evidence_files.md5
echo; echo "Show the stray character is fixed in reference_files.md5:"
grep -C3 22369d5c587517e7ff963c164b878f55  evidence_files.md5
```

27. Create the Hash Datbases. For additional information read the man page for the hfind command and the –i option

```
# Create the Hash Database
hfind -i md5sum reference_files.md5
hfind -i md5sum evidence_files.md5
ls -l
file *
```

28. Create a list of just the MD5 Hashes that were not found in reference_files:

```
awk '{print $1}' evidence_files.md5 | hfind reference_files.md5 | grep \
"Hash Not Found" | awk '{print $1}' > new+changed_hashes_in_evidence.md5
```

29. Create a list of just the MD5 Hashes that were reference_files but are no longer in evidence_files:

```
awk '{print $1}' reference_files.md5 | hfind evidence_files.md5 | grep \
"Hash Not Found" | awk '{print $1}' > missing+changed_hashes_from_evidence.md5
```

30. Display a list of File Sizes using `wc -l *.md5`

31. Determine the file names of the missing+changed files plus the new+changed files

```
grep -f  missing+changed_hashes_from_evidence.md5 reference_files.md5 > missing+changed_files+hashes_from_evidence.md5
grep -f new+changed_hashes_in_evidence.md5 evidence_files.md5 > new+changed_files+hashes_in_evidence.md5
awk '{print $2}' missing+changed_files+hashes_from_evidence.md5 | sed -e 's|/mnt/reference||g' | sort > missing+changed_files_from_evidence.txt
awk '{print $2}' new+changed_files+hashes_in_evidence.md5 | sed -e 's|/mnt/evidence||g' | sort > new+changed_files_in_evidence.txt
```

32. Identify which files are missing, changed, or new. If the file name exists in both the `missing+changed_files_from_evidence.txt` 
and the `new+changed_files_in_evidence.txt` it is a file that has a changed MD5 hash. If the file exists in only the `missing+changed_files_from_evidence.txt` then is was deleted. Conversely, if it exists in only the `new+changed_files_in_evidence.txt` it is a new file.

```
comm -12 missing+changed_files_from_evidence.txt new+changed_files_in_evidence.txt > CHANGED_FILES.txt
comm -13 missing+changed_files_from_evidence.txt new+changed_files_in_evidence.txt > NEW_FILES.txt
comm -23 missing+changed_files_from_evidence.txt new+changed_files_in_evidence.txt > DELETED_FILES.txt
```

33. Determine the Size of each of the files:

```
wc -l CHANGED_FILES.txt NEW_FILES.txt DELETED_FILES.txt 
```

34. Display the reduction ratio:

```
NUMERATOR=$(wc -l CHANGED_FILES.txt NEW_FILES.txt DELETED_FILES.txt | grep total | cut -d" " -f2)
DENOMINATOR=$(wc -l evidence_files.md5 | cut -d" " -f1)
RATIO=$(bc <<< "scale=2; $NUMERATOR / $DENOMINATOR")
echo $NUMERATOR "/" $DENOMINATOR "=" $RATIO
```

35. Which file was deleted?

```
cat DELETED_FILES.txt
```



36. Which files were changed?

```
cat CHANGED_FILES.txt
```

37. Notice that `/etc/crontab` has changed. This may be where malware tries to maintain persistance. Examin it:

```
diff /mnt/reference/etc/crontab /mnt/evidence/etc/crontab
```

38. What is `/bin/wipefs` ?!?

```
strings /mnt/evidence/bin/wipefs
```

Yeah, definitely malicious

39. Notice that `netstat` has a changed MD5 hash. Lets look at the file sizes:

```
ls -l /mnt/evidence/usr/bin/netstat /mnt/reference/usr/bin/netstat
```

Well that is certainly suspicious. Turns out that `netstat` was trojanized!

40. Look at the first 10 files listed in `NEW_FILES.txt`

```
head NEW_FILES.txt
```

41. What is the new `call2mins.sh` file doing in the `grub2` directory?

```
cat /mnt/evidence/boot/grub2/call2mins.sh

cat /mnt/evidence/bin/hello
```

And that is definitely suspicious!

42. Looking again at `NEW_FILES.txt` we can see that there is an AWS credentials file. Lets look at it:

```
cat /mnt/evidence/home/ec2-user/.aws/credentials
```

Eww, not good. Make sure these are revoked.


## Conclusion

I hope that you have enjoyed this demonstration of Differential Filesystem Analysis. Although this demo used AWS EC2, remember that it can be used with any cloud or even on-prem when incremental backups are made.