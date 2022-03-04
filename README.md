解决Mongdb深度分页问题

一. 问题产生

	一个普通订单列表查询，订单数据是存储在mongdb中的，某个用户的订单量比较大，大概有500万单，查询的时候延迟有点高，但还是能接受，但是当用户翻到几十页之后，页面查询直接挂掉了。刚开始以为是业务数据的问题，然后在测试环境调试后发现，不管什么数据翻到几十页都会出现这样的问题，并定位到是使用到 <kbd>mongoTemplate.find</kbd>方法时候出现的，查询一番后知道是深度分页的问题。

	什么是深度分页？就是在进行分页查询时，再查询到符合的全部数据后，再根据页码从第一个数开始做偏移查找，知道找到对应位置的数据，然后再取出相应的数据。当进行大页码的查询时，偏移的数据就会特别大，导致查询mongdb超时报异常。大多数数据库查询都会有这个问题。

二. 分析问题

	问题找到后该如何解决 ，找准问题的根源，大页码数据查询时偏移量导致查询超时，如果不让偏移是不是就能解决问题了。只要让查询的数据刚好是第一个就可以了，完全可以使用前一页的最后一个数据做定位，查询其后面的数据，当然这个数据列是有序的才行，不然查询的数据就是乱的了。事实大多数深度分页的解决办法都是这么做的，也是一个很有效果的方法。

三. 解决问题

	找到解决问题的方法后，再结合自身的业务去分析解决。我们这边是订单查询，mongdb的doc都会有一个主键id，可以自动生成，也可以自己手动生成。我们这边是自己采用雪花算法生成的。雪花算法组成是时间戳+机器码+进程码+唯一码，机器码和进程码可以固定传的，然后唯一码是用来区分同一时间内并发生成的数据，所以整体看来id就是有序且唯一的。注意必须是有序且唯一。所以最后决定采用id做临界值，在进行翻下一页的时候，把当前页的最后一条数据的id值也传给后端，然后只传入每页值的数量就可以了，不要传页码。在进行查询的时候，除了必要的业务参数，还要多加参数就是id，注意，上翻页和下翻页的条件是不相同的。看排序规则，升序排，下翻页就是找比id大的数据，上翻页则相反，降序排，下翻页是找比id小的值，上翻页则相反。这样查出来的数据才是对的，下面我的查询语句，使用的是spring data mongdb

    //下翻页
    if (Objects.equals(pageTurn, 0)) {
        //升序
        if (Objects.equals(sortType, 0)) {
            criteria.and("_id").gt(id);
        } else {
            criteria.and("_id").lt(id);
        }
    } else {
        //升序
        if (Objects.equals(sortType, 0)) {
            criteria.and("_id").lt(id);
        } else {
            criteria.and("_id").gt(id);
        }
    }
    query.addCriteria(criteria);
    query.with(sort).limit(orderPageRequest.getPageSize());

criteria是查询条件，sort是排序条件，limit里面就是每页的数据量了。
