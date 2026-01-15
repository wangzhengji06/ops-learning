## Get history command
`!!` = `Enter`

ctrl + R -> Reverse Search

.bashhistory


## Debug files

cat -A

cat -n

#Just look at the row 33

head  -n 33 | tail -n 1

## Echo

echo -e "" to interpret things like `\n`

echo -n "" mainly used to password transformation

#Color
\033[31 \033[0m 


## Get-help

whereis / whatis / --help / help / man


## File location
/boot -> boot file

/etc -> software

/lib -> shared library

/sys -> system related

/var/log -> logs

/tmp -> temporaryu 

/bin /sbin: binary folder for user and sudo
They usually points to /usr/bin /usr/sbin

/usr/bin /usr/sbin: Same as above

/run: temporary folder for storing 

/dev: hardware device

/misc/cd: used for CD mount if autofs is enabled


## Storage type

df -Th to check the type

ntfs -> windows

fat -> usb

ext4, xfs, swap -> linux


## File type and color


red -> zip

yello -> device

green -> executable

blue -> directory

purple -> socket

light blue -> linked

white -> normal


##  File check and convert

Check: 

file / stat

hexdump / cat -A 

dos2unix # Windows -> Linux


## Link
ln / ln -s / readlink

soflink file has different inode that points to original file's inode

hardlink file has the same inode

folder at least have 2 hardlinks. Why? itself, and .

df -i can check inode usage. Inode usage can reach 100%

## IO

\> / >>

2>&1

sleep 5000 &

cat  > file << eof

cat > file << -eof to cancle the tab at front

tee file # output to terminal also compared to cat, so it is better than cat > file when inisde a pipe



