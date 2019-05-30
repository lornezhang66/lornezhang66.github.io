### 补送话费券拆单sql脚本





> 2019年4月15日09:48:28

```sql
declare
  v_pnum1      varchar2(300);
  v_handlState varchar2(5); --话费券订单处理状态
  v_goodid     varchar2(300);
begin
  /**
  1、订单处理范围：
  取处理订单的范围的逻辑如下：
  时间范围：电视新装从4月2日0点~4.11日 23:04 未拆话费券的宽带电视新装直接受理和预受理的订单，并且渠道为非和微店；
  
  2、处理逻辑
  第一步：插入话费券商品订单ddzx.t_od_goods_order，数据依赖电视的主订单信息；
  第二步：插入商品订单扩展报表，插入活动编号，编号ID固定为：5e3f3cbb-774d-4df9-9a98-c7b26dd32395
  第三步：修改客户订单状态为受理中 05；
  */
  for temp in (select o.cust_order_no,
                      o.channel_code,
                      o.touch_channel_code,
                      g.*
                 from ddzx.t_od_cust_order o, ddzx.t_od_goods_order g
                where o.id = g.cust_order_id
                  and g.crm_serial is not null
                  and g.transact_status in ('01', '02')
                  and o.channel_code <> 'dq_and_league'
                  and g.goods_handle_type in
                      ('BROADBAND_TV_XZ', 'BROADBAND_TV_XZYSL')
                  and g.create_time between
                      to_date('2019-04-02 00:00:01', 'yyyy-mm-dd hh24:mi:ss') and
                      to_date('2019-04-11 23:04:00', 'yyyy-mm-dd hh24:mi:ss')
                  and o.cust_order_no in
                      ('订单编号')
                  and not exists
                (select 1
                         from ddzx.t_od_goods_order t2
                        where t2.cust_order_id = g.cust_order_id
                          and t2.goods_handle_type = 'BROADBAND_BILL')) loop
    --第一步：获取该订单的反冲号码，若反冲号码为空时，取预缴号码，预缴号码为空时，取联系人号码
  
    /**
    订单状态,如果主业务订单状态是完成时,话费券订单状态则为处理中
    */
    v_handlState := '01' ;
    select dbms_random.string('0', 32) into v_goodid from dual;
    --取反冲号码 
  
    begin
      select t.attr_value
        into v_pnum1
        from ddzx.t_od_goods_order_attr t,
             ddzx.t_od_cust_order       o,
             ddzx.t_od_goods_order      g
       where g.cust_order_id = o.id
         and g.id = t.goods_order_id
         and t.attr_code = 'prerechargenum'
         and o.cust_order_no = temp.cust_order_no;
    exception
      when others then
        begin
          if v_pnum1 is null then
            select nvl(t.attr_value, '')
              into v_pnum1
              from ddzx.t_od_goods_order_attr t,
                   ddzx.t_od_cust_order       o,
                   ddzx.t_od_goods_order      g
             where g.cust_order_id = o.id
               and g.id = t.goods_order_id
               and t.attr_name = '预缴号码'
               and o.cust_order_no = temp.cust_order_no;
          else
            v_pnum1 := temp.CONT_BILLID;
          end if;
        exception
          when others then
            v_pnum1 := temp.CONT_BILLID;
          
        end;
      
    end;
  
    --执行插入语句
    insert into ddzx.t_od_goods_order
      (ID,
       CUST_ORDER_ID,
       ORDER_BILLID,
       IDEMPOTENT_ID,
       GOODS_ID,
       GOODS_NAME,
       GOODS_UNIT_PRICE,
       GOODS_NUM,
       GOODS_TYPE,
       GOODS_HANDLE_TYPE,
       GOODS_DESC,
       SHOULD_MONEY_TOTAL,
       DISTCOUNT_MONEY_TOTAL,
       FACT_MONEY_TOTAL,
       TRANSACT_ACCOUNT,
       CITY,
       COUNTY,
       TRANSACT_STATUS,
       TRANSACT_FAILURE_REASON,
       BUSI_CENTER_CODE,
       GOODS_STATUS,
       IS_MAIN,
       EXEC_INDEX,
       TRANSACT_TYPE,
       HANDLE_INTERVAL,
       HANDLE_RETRY_NUM,
       HANDLE_STATUS,
       HANDLE_TIME_BEGIN,
       HANDLE_TIME_END,
       HANDLED_COUNT,
       HANDLE_TIME_NEXT,
       CREATE_TIME,
       UPDATE_TIME,
       REMARK,
       IS_DELETE,
       CUST_CERT_CODE,
       CRM_SERIAL,
       CRM_STATE,
       CONT_BILLID,
       CONT_NAME,
       ACCOUNT_ID,
       CRM_COMPLET_TIME,
       PREPAY_BILLID)
    values
      (v_goodid,
       temp.cust_order_id,
       temp.order_billid,
       (select dbms_random.string('0', 36) from dual),
       'DZ0915',
       '宽带和电视新装送30元话费券',
       null,
       null,
       'BROADBAND',
       'BROADBAND_BILL',
       null,
       null,
       null,
       null,
       v_pnum1,
       null,
       null,
       '00',
       null,
       'BROADBAND',
       null,
       '0',
       2,
       '2',
       600,
       1,
       v_handlState,
       null,
       null,
       0,
       sysdate - 0.1,
       temp.create_time,
       temp.create_time,
       null,
       0,
       null,
       null,
       null,
       null,
       null,
       null,
       null,
       null);
  
    --第二部 插入商品扩展属性表，用于限定活动
    insert into ddzx.t_od_goods_order_attr
      (ID,
       GOODS_ORDER_ID,
       ORDER_BILLID,
       ATTR_CODE,
       ATTR_NAME,
       ATTR_VALUE,
       CREATE_TIME,
       UPDATE_TIME,
       IS_DELETE)
    values
      ((select dbms_random.string('0', 32) from dual),
       v_goodid,
       temp.order_billid,
       'activityId',
       '活动ID',
       '5e3f3cbb-774d-4df9-9a98-c7b26dd32395',
       sysdate,
       sysdate,
       '0');
  
    --billCode 话费券 DZ0915
    insert into ddzx.t_od_goods_order_attr
      (ID,
       GOODS_ORDER_ID,
       ORDER_BILLID,
       ATTR_CODE,
       ATTR_NAME,
       ATTR_VALUE,
       CREATE_TIME,
       UPDATE_TIME,
       IS_DELETE)
    values
      ((select dbms_random.string('0', 32) from dual),
       v_goodid,
       temp.order_billid,
       'billCode',
       '话费券',
       'DZ0915',
       sysdate,
       sysdate,
       '0');
  
    --第三步 修改已完结的客户订单状态为受理中 05
    if temp.transact_status = '02' then
      update ddzx.t_od_cust_order
         set status = '05'
       where cust_order_no = temp.cust_order_no
         and status = '99';
    end if;
    commit;
  end loop;
  
end;
```

