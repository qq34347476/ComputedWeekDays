# ComputedWeekDays
最近一个项目要求推演延顺工期，分享出啦源码， 可以根据起始时间推演去除节假日，特定调休后工作几天是几号，也可反推两个时间段中的工作日
```javascript
//  假日
let Holiday = ['2020-01-01','2020-01-24', '2020-01-25', '2020-01-26', '2020-01-27', '2020-01-28', '2020-01-29', '2020-01-30', '2020-04-04', '2020-04-05', '2020-04-06', '2020-05-01', '2020-05-02', '2020-05-03', '2020-06-25', '2020-06-26', '2020-06-27', '2020-10-01', '2020-10-02', '2020-10-03', '2020-10-04', '2020-10-05', '2020-10-06', '2020-10-07']

// 调休加班补班
let WeekendsOff = []

/**
 * 计算当前时间（或指定时间），向前推算周数(weekcount)，得到结果周的第一天的时期值；
 * @param {String} mode 推算模式（'cn'表示国人习惯【周一至周日】；'en'表示国际习惯【周日至周一】）
 * @param {Number} weekcount 表示周数（0-表示本周， 1-前一周，2-前两周，以此推算）；
 * @param {String} end 指定时间的字符串（未指定则取当前时间）
 */
function nearlyWeeks(mode = 'cn', weekcount = 0, end = new Date()) {
    end = new Date(new Date(end).toDateString())
    let days = 0
    if (mode === 'cn') //中国时间模式
        days = (new Date(end).getDay() == 0 ? 7 : new Date(end).getDay()) - 1
    else //歪果仁时间模式
        days = new Date(end).getDay()
    return new Date(new Date(end).getTime() - (days + weekcount * 7) * 24 * 60 * 60 * 1000)
}

/**
 * 计算一段时间内工作的天数。不包括周末和法定节假日，法定调休日为工作日，周末为周六、周日两天
 * @param {String} beginDay 时间段开始日期
 * @param {String} endDay 时间段结束日期
 * @param {String} mode 推算模式（'cn'表示国人习惯【周一至周日】；'en'表示国际习惯【周日至周一】）
 */
function getWorkDayCount(beginDay, endDay, mode = 'cn') {
    let begin = new Date(new Date(beginDay).toDateString())
    let end = new Date(new Date(endDay).toDateString())
    //每天的毫秒总数，用于以下换算
    const daytime = 24 * 60 * 60 * 1000;
    //两个时间段相隔的总天数
    const days = (end - begin) / daytime + 1;
    //时间段起始时间所在周的第一天
    const beginWeekFirstDay = nearlyWeeks(mode, 0, new Date(beginDay).getTime()).getTime();
    //时间段结束时间所在周的最后天
    const endWeekOverDay = nearlyWeeks(mode, 0, new Date(endDay).getTime()).getTime() + 6 * daytime;
    //由beginWeekFirstDay和endWeekOverDay换算出，周末的天数
    let weekEndCount = ((endWeekOverDay - beginWeekFirstDay) / daytime + 1) / 7 * 2;
    //根据参数mode，调整周末天数的值
    if (mode == 'cn') { //中国
        //周一到周五  结束
        if (new Date(endDay).getDay() > 0 && new Date(endDay).getDay() < 6)  weekEndCount -= 2
        //周6  结束
        if (new Date(endDay).getDay() == 6) weekEndCount -= 1
        //周日  开始
        if (new Date(beginDay).getDay() == 0) weekEndCount -= 1 
    } else { //歪果仁
        if (new Date(beginDay).getDay() < 6) weekEndCount -= 1;
        if (new Date(beginDay).getDay() > 0) weekEndCount -= 1;
    }
    //根据调休设置，调整周末天数（排除调休日）
    WeekendsOff.forEach(offitem => {
        let itemDay = new Date(offitem.split('-')[0] + '/' + offitem.split('-')[1] + '/' + offitem.split('-')[2])
        //如果调休日在时间段区间内，且为周末时间（周六或周日），周末天数值-1
        if (itemDay.getTime() >= begin.getTime() && itemDay.getTime() <= end.getTime() && (itemDay.getDay() == 0 || itemDay.getDay() == 6)) weekEndCount -= 1
    })
    //根据法定假日设置，计算时间段内周末的天数（包含法定假日）
    Holiday.forEach(itemHoliday => {
        let itemDay = new Date(itemHoliday.split('-')[0] + '/' + itemHoliday.split('-')[1] + '/' + itemHoliday.split('-')[2])
        //如果法定假日在时间段区间内，且为工作日时间（周一至周五），周末天数值+1
        if (itemDay.getTime() >= begin.getTime() && itemDay.getTime() <= end.getTime() && itemDay.getDay() > 0 && itemDay.getDay() < 6) weekEndCount += 1
    })
    //工作日 = 总天数 - 周末天数（包含法定假日并排除调休日）
    return parseInt(days - weekEndCount)
};

/**
 * 计算工期结束日期
 * @param {Number} howLong 工期多少天
 * @param {Boolean} mode 周末是否加班[默认false,周末不算工期]
 * @param {String} beginDay 开始日期
 */
function getEndConstructionTime(howLong = 1, mode = false, beginDay = new Date()){
    let count = howLong //工作日
    //每天的毫秒总数，用于以下换算
    const daytime = 24 * 60 * 60 * 1000;
    let res = 0
    function constructionPeriodIsOver(howLong = 1, mode = false, beginDay = new Date()){
        //结束日期
        let end = (howLong - 1) * daytime + new Date(beginDay).getTime()
        //周末节假日不算工期
        if(mode) {
            workDay = getWorkDayCount(new Date(beginDay),new Date(end))
            if(workDay >= count){ //工期结束[实际工作天数 >= 工期天数]
                return 
            }else{ //没结束
                res = (count + (howLong - workDay))
                constructionPeriodIsOver(res,true)
            }
        }else{
            return new Date(end)
        }
    }
    constructionPeriodIsOver(count, mode, beginDay)
    return new Date((res - 1) * daytime + new Date(beginDay).getTime())
}
console.log(getEndConstructionTime(10,true));
```
