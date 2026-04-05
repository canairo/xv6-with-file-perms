version of xv6 reimplemented with file permissions. why? because i can

key changes and nuances:

we need to change two things, the `dinode` and `inode` struct, to add a new field `permissions`:

```c
// in-memory copy of an inode
struct inode {
  uint dev;           // Device number
  uint inum;          // Inode number
  int ref;            // Reference count
  struct sleeplock lock; // protects everything below here
  int valid;          // inode has been read from disk?

  short type;         // copy of disk inode
  short major;
  short minor;
  short nlink;
  short permissions; // <- js add this

  uint size;
  uint addrs[NDIRECT+1];
};
```

```c
// On-disk inode structure
struct dinode {
  short type;           // File type
  short major;          // Major device number (T_DEVICE only)
  short minor;          // Minor device number (T_DEVICE only)
  short nlink;          // Number of links to inode in file system
  short permissions;    // file perms
  uint size;            // Size of file (bytes)
  uint addrs[NDIRECT+1];   // Data block addresses
};
```

bc of abstruse padding issues `dinode` needs 2 be exactly 64 bytes, we make do with this by reducing `NDIRECT` by 1 so that it all fits (xd)

then we need to change the `create()` function but also funnily enough the `iupdate()` and `iunlockput()` functions

```c
void
iupdate(struct inode *ip)
{
  struct buf *bp;
  struct dinode *dip;

  bp = bread(ip->dev, IBLOCK(ip->inum, sb));
  dip = (struct dinode*)bp->data + ip->inum%IPB;
  dip->type = ip->type;
  dip->major = ip->major;
  dip->minor = ip->minor;
  dip->nlink = ip->nlink;
  dip->size = ip->size;
  dip->permissions = ip->permissions;
  memmove(dip->addrs, ip->addrs, sizeof(ip->addrs));
  log_write(bp);
  brelse(bp);
}
```

for the first time ever we also have to look at the `mkfs` dir bc thats where all the files are initialised

```c
uint
ialloc(ushort type)
{
  uint inum = freeinode++;
  struct dinode din;

  bzero(&din, sizeof(din));
  din.type = xshort(type);
  din.nlink = xshort(1);
  din.size = xint(0);
  din.permissions = xshort(0);
  winode(inum, &din);
  return inum;
}
```

and finally we can introduce a new syscall sys_chmod to edit the permissions flag of an inode

```c
uint64
sys_chmod(void)
{
  
  int permissions;
  char path[MAXPATH];
  int n;
  struct inode *ip;
  
  argint(1, &permissions);
  if((n = argstr(0, path, MAXPATH)) < 0)
    return -1;

  begin_op();

  if((ip = namei(path)) == 0){
    end_op();
    return -1;
  }
    
  ilock(ip);
  ip->permissions = permissions;
  iupdate(ip);
  iunlockput(ip);
  end_op();
  return 0;
}

```

ofc this necessitates all the syscall scaffolding that you need to do anyway
