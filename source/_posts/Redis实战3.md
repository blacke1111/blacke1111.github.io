---
title: Redis实战3
date: 2022-05-25 17:27:22
categories: Redis
---

# 达人探店

## 发布探店笔记

发布探店笔记
探店笔记类似点评网站的评价，往往是图文结合。对应的表有两个：
• tb_blog：探店笔记表，包含笔记中的标题、文字、图片等
• tb_blog_comments：其他用户对探店笔记的评价

![image-20220525202020483](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220525202020483.png)

点赞

需求：

* 同一个用户只能点赞一次，再次点击则取消点赞
* 如果当前用户已经点赞，则点赞按钮高亮显示（前端已实现，判断字段Blog类的isLike属性）
  实现步骤：
  ① 给Blog类中添加一个isLike字段，标示是否被当前用户点赞
  ② 修改点赞功能，利用Redis的set集合判断是否点赞过，未点赞过则点赞数+1，已点赞过则点赞数-1
  ③ 修改根据id查询Blog的业务，判断当前登录用户是否点赞过，赋值给isLike字段
  ④ 修改分页查询Blog业务，判断当前登录用户是否点赞过，赋值给isLike字段

```java
 @Override
    public Result queryBlogById(Long id) {

        //1 查询blog
        Blog blog = getById(id);
        if (blog==null){
            return Result.fail("笔记不存在!");
        }
        //2 查询blog有关的用户
        queryBlogUser(blog);
        //3.查询blog是否被点赞
        blog.setIsLike(BooleanUtil.isTrue(isBlogLiked(blog)));
        return Result.ok(blog);
    }

    private Boolean isBlogLiked(Blog blog) {
        //1 获取登录用户
        Long userId=UserHolder.getUser().getId();
        if (userId!=null) {
            //2.判断当前登录用户是否已经点赞
            String key = BLOG_LIKED_KEY + blog.getId();
            return stringRedisTemplate.opsForSet().isMember(key, userId.toString());
        }
        return false;
    }

    @Override
    public Result likeBlog(Long id) {
        //1 获取登录用户
        Long userId=UserHolder.getUser().getId();
        //2.判断当前登录用户是否已经点赞
        String key = BLOG_LIKED_KEY + id;
        Boolean isLiked = stringRedisTemplate.opsForSet().isMember(key, userId.toString());
        if (BooleanUtil.isFalse(isLiked)){
            //3 未点赞,可以点赞
            //3.1 数据库点赞数 + 1
            boolean isSuccess = update().setSql("liked=liked+1").eq("id", id).update();
            //3.2 保存用户到redis的set集合
            if (isSuccess) {
                stringRedisTemplate.opsForSet().add(key, userId.toString());
            }
        }else {
            //4如果已点赞，取消点赞
            //4.1 数据库点赞数-1
            boolean isSuccess = update().setSql("liked=liked-1").eq("id", id).update();
            if (isSuccess) {
                //4.2 把用户从redis集合中移除
                stringRedisTemplate.opsForSet().remove(key, userId.toString());
            }
        }

        return Result.ok();
    }

    private void queryBlogUser(Blog blog) {
        Long userId = blog.getUserId();
        User user = userService.getById(userId);
        blog.setIcon(user.getIcon());
        blog.setName(user.getNickName());
    }
```



## 点赞排行榜

需求：按照点赞时间先后排序，返回Top5的用户

![image-20220531182121411](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220531182121411.png)

改造点赞代码实现SortedSet

```java
 @Override
    public Result likeBlog(Long id) {
        //1 获取登录用户
        Long userId=UserHolder.getUser().getId();
        //2.判断当前登录用户是否已经点赞
        String key = BLOG_LIKED_KEY + id;
        Double score = stringRedisTemplate.opsForZSet().score(key, userId.toString());
        if (score ==null){
            //3 未点赞,可以点赞
            //3.1 数据库点赞数 + 1
            boolean isSuccess = update().setSql("liked=liked+1").eq("id", id).update();
            //3.2 保存用户到redis的set集合 zadd key value score
            if (isSuccess) {
                stringRedisTemplate.opsForZSet().add(key, userId.toString(),System.currentTimeMillis());
            }
        }else {
            //4如果已点赞，取消点赞
            //4.1 数据库点赞数-1
            boolean isSuccess = update().setSql("liked=liked-1").eq("id", id).update();
            if (isSuccess) {
                //4.2 把用户从redis集合中移除
                stringRedisTemplate.opsForSet().remove(key, userId.toString());
            }
        }

        return Result.ok();
    }
```

排行榜：

```java
 @Override
    public Result queryBlogLikes(Long id) {
        //实现top5的点赞用户 zrange key 0 4
        String key = BLOG_LIKED_KEY + id;
        Set<String> top5 = stringRedisTemplate.opsForZSet().range(key, 0, 4);
        if (top5==null || top5.isEmpty()){
            return Result.ok(Collections.emptyList());
        }
        //2 解析出其中的用户id
        //因为数据库使用的in 查询 用户 导致userDTOS顺序乱了
        List<Long> ids = top5.stream().map(Long::valueOf).collect(Collectors.toList());
        String idStr = StrUtil.join(",", ids);
        List<UserDTO> userDTOS =
                userService.query().in("id",ids).last("ORDER BY FIELD(id,"+idStr+")").list()
                .stream().map(user ->
                BeanUtil.copyProperties(user, UserDTO.class)
        ).collect(Collectors.toList());


        //3 更具用户id查询用户
        return Result.ok(userDTOS);
    }
```

## 关注功能:

![image-20220531185439294](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220531185439294.png)

实现关注和取关功能

需求：基于该表数据结构，实现两个接口：
① 关注和取关接口
② 判断是否关注的接口
关注是User之间的关系，是博主与粉丝的关系，数据库中有一张tb_follow表来标示：

![image-20220531185506055](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220531185506055.png)

注意: 这里需要把主键修改为自增长，简化开发。

```java
@Override
    public Result follow(Long followUserId, Boolean isFollow) {
        //1 获取登录用户
        Long userId = UserHolder.getUser().getId();
        //1 判断时关注还是取关
        if (isFollow){
            //2关注 新增数据
            Follow follow = new Follow();
            follow.setUserId(userId);
            follow.setFollowUserId(followUserId);
            save(follow);
        }else {
            //3 取关 ，删除
            QueryWrapper<Follow> wrapper = new QueryWrapper<>();
            wrapper.eq("user_id",userId);
            wrapper.eq("follow_user_id",followUserId);
            remove(wrapper);
        }
        return Result.ok();
    }

    @Override
    public Result isFollow(Long followUserId) {
        Long userId = UserHolder.getUser().getId();
        //1 查询 是否关注
        QueryWrapper<Follow> wrapper = new QueryWrapper<>();
        wrapper.eq("user_id",userId);
        wrapper.eq("follow_user_id",followUserId);
        int count = count(wrapper);
        return Result.ok(count>0);
    }
```

## 共同关注：

**利用Set集合sinter 命令求交集 实现共同关注**

改造关注接口：

```java
@Override
    public Result follow(Long followUserId, Boolean isFollow) {
        //1 获取登录用户
        Long userId = UserHolder.getUser().getId();
        //1 判断时关注还是取关
        if (isFollow){
            //2关注 新增数据
            Follow follow = new Follow();
            follow.setUserId(userId);
            follow.setFollowUserId(followUserId);
            boolean isSuccess = save(follow);
            if (isSuccess){
                // 把关注用户的id,放入redis的set集合
                String key="follows:"+userId;
                stringRedisTemplate.opsForSet().add(key,followUserId.toString());
            }
        }else {
            //3 取关 ，删除
            QueryWrapper<Follow> wrapper = new QueryWrapper<>();
            wrapper.eq("user_id",userId);
            wrapper.eq("follow_user_id",followUserId);
            boolean isSuccess = remove(wrapper);
            if (isSuccess){
                stringRedisTemplate.opsForSet().remove("follows:"+userId,followUserId.toString());
            }
        }
        return Result.ok();
    }
```

```java
@Override
    public Result followCommons(Long id) {
        //1 获取当前关注
        Long userId = UserHolder.getUser().getId();
        String key1 = "follows:" + userId;
        String key2="follows:"+id;
        //求 交集
        Set<String> intersect = stringRedisTemplate.opsForSet().intersect(key1, key2);
        if (intersect ==null || intersect.isEmpty()){
            return Result.ok(Collections.emptyList());
        }
        List<Long> ids = intersect.stream().map(Long::valueOf).collect(Collectors.toList());
        //4 查询用户
        List<UserDTO> userDTOS = userService.listByIds(ids).stream().map(user -> BeanUtil.copyProperties(user, UserDTO.class)).collect(Collectors.toList());
        return Result.ok(userDTOS);
    }
```

## 关注推送

![image-20220531195030787](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220531195030787.png)

![image-20220531195036737](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220531195036737.png)

### Feed模式

Feed流产品有两种常见模式：

* Timeline：不做内容筛选，简单的按照内容发布时间排序，常用于好友或关注。例如朋友圈

  > 优点：信息全面，不会有缺失。并且实现也相对简单
  >  缺点：信息噪音较多，用户不一定感兴趣，内容获取效率低

* 智能排序：利用智能算法屏蔽掉违规的、用户不感兴趣的内容。推送用户感兴趣信息来吸引用户

  > 优点：投喂用户感兴趣信息，用户粘度很高，容易沉迷
  >  缺点：如果算法不精准，可能起到反作用

本例中的个人页面，是基于关注的好友来做Feed流，因此采用Timeline的模式。该模式的实现方案有三种：
① 拉模式
② 推模式
③ 推拉结合

Feed流的实现方案1

拉模式：也叫做读扩散。

![image-20220531195654024](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220531195654024.png)

Feed流的实现方案2

推模式：也叫做写扩散

![image-20220531200136147](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220531200136147.png)

Feed流的实现方案3
推拉结合模式：也叫做读写混合，兼具推和拉两种模式的优点。

![image-20220531201156408](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220531201156408.png)

![image-20220531201209226](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220531201209226.png)

基于推模式实现关注推送功能

需求：
① 修改新增探店笔记的业务，在保存blog到数据库的同时，推送到粉丝的收件箱
② 收件箱满足可以根据时间戳排序，必须用Redis的数据结构实现
③ 查询收件箱数据时，可以实现分页查询

![image-20220531201634633](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220531201634633.png)

Feed流的分页问题

Feed流中的数据会不断更新，所以数据的角标也在变化，因此不能采用传统的分页模式。

![image-20220531202228697](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220531202228697.png)

Feed流的滚动分页

Feed流中的数据会不断更新，所以数据的角标也在变化，因此不能采用传统的分页模式。

![image-20220531202335436](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220531202335436.png)

推送功能的实现:

```java
//follow_user_id 是被关注的up主的id
@Override
    public Result saveBlog(Blog blog) {
        //1获取登录用户
        UserDTO userDTO =UserHolder.getUser();
        blog.setUserId(userDTO.getId());

        boolean isSuccess = save(blog);
        if (!isSuccess){
            return  Result.fail("新增笔记失败！");
        }
        //3 查询笔记作者的所有粉丝select * from tb_follow where follow_user_id=?
        List<Follow> follows = followService.query().eq("follow_user_id", userDTO.getId()).list();

        //4 推送笔记作者的所有粉丝
        for (Follow follow : follows) {
            //4.1 获取粉丝 id
            Long userId = follow.getUserId();
            String key=FEED_KEY+userId;
            stringRedisTemplate.opsForZSet().add(key,blog.getId().toString(),System.currentTimeMillis());
        }
        //5 返回id
        return Result.ok(blog.getId());
    }
```

滚动分页：

```java
 @Override
    public Result queryBlogOfFollow(Long max, Integer offset) {
        //1。 找到收件箱
        Long userId = UserHolder.getUser().getId();
        String key=FEED_KEY+userId;
        //2 查询收件箱 ZRANGEBYSCORE key max min limit offset count
        Set<ZSetOperations.TypedTuple<String>> typedTuples = stringRedisTemplate.opsForZSet().
                reverseRangeByScoreWithScores(key, 0, max, offset, 2);
        if (typedTuples==null|| typedTuples.isEmpty()){
            return Result.ok();
        }
        List<Long> ids=new ArrayList<>(typedTuples.size());
        long mintime=0;
        int os=1;
        //3 解析数据 blogId,score(时间戳),offset(和时间戳一样的数据的个数)
        for (ZSetOperations.TypedTuple<String> typedTuple : typedTuples) {
            //获取id
            ids.add(Long.valueOf(typedTuple.getValue()));
            // 时间戳
             long time = typedTuple.getScore().longValue();
             if (time==mintime){
                 os++;
             }else {
                 mintime=time;
                 os=1;
             }
        }
        String idStr = StrUtil.join(",", ids);
        //4 根据id查询blog
        List<Blog> blogs = query().in("id",ids).last("ORDER BY FIELD(id,"+idStr+")").list();
        for (Blog blog : blogs) {
            //2 查询blog有关的用户
            queryBlogUser(blog);
            //3.查询blog是否被点赞
            blog.setIsLike(BooleanUtil.isTrue(isBlogLiked(blog)));
        }
        ScrollResult scrollResult = new ScrollResult();
        scrollResult.setList(blogs);
        scrollResult.setMinTime(mintime);
        scrollResult.setOffset(os);
        return Result.ok(scrollResult);
    }
```

# 附近商户

## 附近商户搜索
