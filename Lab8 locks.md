# Lab8: locks

## Memory allocator

```c
// kernel/kalloc.c
void
kinit()
{
  char c[5];
  for(int i = 0; i < NCPU; i++){
  	snprintf(c, 5, "kmem%d", i);
  	initlock(&kmem[i].lock, c);
  }
  freerange(end, (void*)PHYSTOP);
}
void
kfree(void *pa)
{
  struct run *r;

  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    panic("kfree");

  // Fill with junk to catch dangling refs.
  memset(pa, 1, PGSIZE);
  push_off();
  int cpuindex = cpuid();
  pop_off();
  r = (struct run*)pa;
  acquire(&kmem[cpuindex].lock);
  r->next = kmem[cpuindex].freelist;
  kmem[cpuindex].freelist = r;
  release(&kmem[cpuindex].lock);
}
void *
kalloc(void)
{
  struct run *r;
  push_off();
  int cpuindex = cpuid();
  pop_off();
  acquire(&kmem[cpuindex].lock);
  r = kmem[cpuindex].freelist;
  if(r)
    kmem[cpuindex].freelist = r->next;
  release(&kmem[cpuindex].lock);
  if(r)
    memset((char*)r, 5, PGSIZE); // fill with junk
  else{
      for(int i = 0; i < NCPU; i++){
  	acquire(&kmem[i].lock);
  	r = kmem[i].freelist;
  	if(r)
  	  kmem[i].freelist = r -> next;
  	release(&kmem[i].lock);
  	if(r){
  	  memset((char*)r, 5, PGSIZE);
  	  break;
  	}
     }
  }
  return (void*)r;
}
```

## Buffer cache

```c
// kernel/bio.c
#define NBUCKET 13
struct {
  struct spinlock lock;
  struct spinlock bucketlock[NBUCKET];
  struct buf buf[NBUF];

  // Linked list of all buffers, through prev/next.
  // Sorted by how recently the buffer was used.
  // head.next is most recent, head.prev is least.
  //struct buf head;
  struct buf head[NBUCKET];
} bcache;
void
binit(void)
{
  initlock(&bcache.lock, "bcache");
  char c[8];
  for(int i = 0; i < NBUCKET; i++){
        snprintf(c, 8, "bcache%d", i);
  	initlock(&bcache.bucketlock[i], c);
  }
  // Create linked list of buffers
  for(int i = 0; i < NBUCKET; i++){
  	bcache.head[i].prev = &bcache.head[i];
  	bcache.head[i].next = &bcache.head[i];	
  }
  for(int i = 0; i < NBUF; i++){
    struct buf* b = bcache.buf + i;
    int index = i % NBUCKET;
    b->timestamp = ticks;
    b->next = bcache.head[index].next;
    b->prev = &bcache.head[index];
    initsleeplock(&b->lock, "buffer");
    bcache.head[index].next->prev = b;
    bcache.head[index].next = b;
  }
}
static struct buf*
bget(uint dev, uint blockno)
{
  struct buf *b;

  //acquire(&bcache.lock);

  // Is the block already cached?
  int index = blockno % NBUCKET;
  acquire(&bcache.bucketlock[index]);
  for(b = bcache.head[index].next; b != &bcache.head[index]; b = b->next){
    if(b->dev == dev && b->blockno == blockno){
      b->refcnt++;
      b->timestamp = ticks;
      release(&bcache.bucketlock[index]);
      //release(&bcache.lock);
      acquiresleep(&b->lock);
      return b;
    }
  }
  release(&bcache.bucketlock[index]);
  // Not cached.
  // Recycle the least recently used (LRU) unused buffer.
  acquire(&bcache.lock);
  int label = -1;
  int OK = 0;
  struct buf* eviction = &bcache.head[0];
  for(int i = 0; i < NBUCKET; i++){
  acquire(&bcache.bucketlock[i]);
  for(b = bcache.head[i].prev; b != &bcache.head[i]; b = b->prev){
    if(b->refcnt == 0) {
      if(OK == 0 || b->timestamp < eviction->timestamp){
        OK = 1;
        if(label != -1)
          release(&bcache.bucketlock[label]);
        label = i;
        eviction = b;
      }
      break;
    }
  }
  if(label != i){
  	release(&bcache.bucketlock[i]);
  }
 }
  if(!OK)
  	panic("bget: no buffers");
  b = eviction;
  if(label != index)
  	acquire(&bcache.bucketlock[index]);
  b->prev->next = b->next;
  b->next->prev = b->prev;
  if(label != index)
  	release(&bcache.bucketlock[label]);
  b->prev = &bcache.head[index];
  b->next = bcache.head[index].next;
  bcache.head[index].next = b;
  b->next->prev = b;
  b->dev = dev;
  b->blockno = blockno;
  b->valid = 0;
  b->refcnt = 1;
  release(&bcache.bucketlock[index]);
  release(&bcache.lock);
  acquiresleep(&b->lock);
  return b;
  
}
void
brelse(struct buf *b)
{
  if(!holdingsleep(&b->lock))
    panic("brelse");
  int index = b->blockno % 13;
  releasesleep(&b->lock);
  acquire(&bcache.bucketlock[index]);
  b->refcnt--;
  if (b->refcnt == 0) {
    // no one is waiting for it.
    b->next->prev = b->prev;
    b->prev->next = b->next;
    b->next = bcache.head[index].next;
    b->prev = &bcache.head[index];
    bcache.head[index].next->prev = b;
    bcache.head[index].next = b;
    b->timestamp = ticks;
  }
  
  release(&bcache.bucketlock[index]);
}
```

