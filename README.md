# Nordic 中心设备添加自定义服务处理



照搬`ble_nus_c.c`和`ble_nus_c.h`内容来完成自定义服务的处理。这里中心设备我采用的工程例子是 `ble_app_uart_c`。



将`components\ble\ble_services\ble_nus_c`中的 `ble_nus_c.c`和`ble_nus_c.h` 拷贝到我们的工程中。

修改文件名，避免文件重定义问题。然后就是依样画葫芦，照着 `ble_nus_c.c` 和 `ble_nus_c.h` 进行修改。

这里[修改好的文件](https://github.com/duapple/nordic_ble_central_custom_services/tree/master)我放在 `github`上。

这里直接把所有的`nus`名字换成我们自定义的。

`uuid` 也换成自定义的，这里的三个 `uuid` 需要是连续的。

```c
#define MYSERVICE_BASE_UUID                   {{0x66, 0xCA, 0xDC, 0x24, 0x0E, 0xE5, 0xA9, 0xE0, 0x93, 0xF3, 0xA3, 0xB5, 0x00, 0x00, 0x40, 0x6E}} /**< Used vendor-specific UUID. */

#define BLE_UUID_MYSERVICE_SERVICE            0x2221                      /**< The UUID of the Nordic UART Service. */
#define BLE_UUID_MYSERVICE_RX_CHARACTERISTIC  0X2222                      /**< The UUID of the RX Characteristic. */
#define BLE_UUID_MYSERVICE_TX_CHARACTERISTIC  0x2223                      /**< The UUID of the TX Characteristic. */
```



然后在 `main` 中对应的位置，添加我们自定义的服务处理。这里照着 `nus` 服务的代码来进行修改。

先定义服务实例。

```c
BLE_MYSERVICE_C_DEF(m_ble_myservice_c);
```

然后定义服务初始化接口。

```c
static void myservice_c_init(void)
{
    ret_code_t       err_code;
    ble_myservice_c_init_t init;

    init.evt_handler   = ble_myservice_c_evt_handler;
    init.error_handler = nus_error_handler;
    init.p_gatt_queue  = &m_ble_gatt_queue;

    err_code = ble_myservice_c_init(&m_ble_myservice_c, &init);
	if (err_code != NRF_SUCCESS)
	{
		NRF_LOG_INFO("[%s] err_code: %d", __func__, err_code);
	}
    APP_ERROR_CHECK(err_code);
}
```

这里的服务初始化接口可能会报错。错误代码：`0x04` ，原因：`NRF_ERROR_NO_MEM` 。

解决方案：

在 `sdk_config.h` 中为我们自定义的 `uuid` 分配内存空间。

<div align=center><img src="https://raw.githubusercontent.com/duapple/Code/master/pic/20210127105458.png" alt="uuid_count" style="zoom:50%;" width="60%" high="60%"/>

运行时，根据提示`RTT View`打印修改 `RAM` 大小。

<div align=center><img src="https://raw.githubusercontent.com/duapple/Code/master/pic/20210127105537.png" alt="RAM" style="zoom:50%;" width="60%" high="60%" />



`void bsp_event_handler(bsp_event_t event)` 事件中添加 断连处理。

```c
void bsp_event_handler(bsp_event_t event)
{
    ret_code_t err_code;

    switch (event)
    {
    case BSP_EVENT_DISCONNECT:

        err_code = sd_ble_gap_disconnect(m_ble_nus_c.conn_handle, \
                                         BLE_HCI_REMOTE_USER_TERMINATED_CONNECTION);
#if ENABLE_BLE_MYSERVICE
        err_code = sd_ble_gap_disconnect(m_ble_myservice_c.conn_handle, \
                                         BLE_HCI_REMOTE_USER_TERMINATED_CONNECTION);
#endif
        if (err_code != NRF_ERROR_INVALID_STATE)
        {
            APP_ERROR_CHECK(err_code);
        }
        break;
}
```



`static void ble_evt_handler(ble_evt_t const * p_ble_evt, void * p_context)` 中添加 连接处理。

经测试，这里不用添加也是可以的。

```c
static void ble_evt_handler(ble_evt_t const * p_ble_evt, void * p_context)
{
    ret_code_t            err_code;
    ble_gap_evt_t const * p_gap_evt = &p_ble_evt->evt.gap_evt;

    switch (p_ble_evt->header.evt_id)
    {
    case BLE_GAP_EVT_CONNECTED:
        // NRF_LOG_INFO("BLE_GAP_EVT_CONNECTED");

        err_code = ble_nus_c_handles_assign(&m_ble_nus_c, p_ble_evt->evt.gap_evt.conn_handle, NULL);
#if ENABLE_BLE_MYSERVICE
        err_code = ble_myservice_c_handles_assign(&m_ble_myservice_c, p_ble_evt->evt.gap_evt.conn_handle, NULL);
#endif
        APP_ERROR_CHECK(err_code);

        err_code = bsp_indication_set(BSP_INDICATE_CONNECTED);
        APP_ERROR_CHECK(err_code);

        // start discovery of services. The NUS Client waits for a discovery result
        err_code = ble_db_discovery_start(&m_db_disc, p_ble_evt->evt.gap_evt.conn_handle);
        APP_ERROR_CHECK(err_code);
        break;
}
```



定义自定义服务的事件处理回调。照着 `nus` 服务的进行修改就好了。

```c
static void ble_myservice_c_evt_handler(ble_myservice_c_t * p_ble_myservice_c, ble_myservice_c_evt_t const * p_ble_myservice_evt)
{
	DEBUG_INFO();
    ret_code_t err_code;

    switch (p_ble_myservice_evt->evt_type)
    {
    case BLE_NUS_C_EVT_DISCOVERY_COMPLETE:
        NRF_LOG_INFO("Myservice Discovery complete.");
        err_code = ble_myservice_c_handles_assign(p_ble_myservice_c, p_ble_myservice_evt->conn_handle, &p_ble_myservice_evt->handles);
        APP_ERROR_CHECK(err_code);

        err_code = ble_myservice_c_tx_notif_enable(p_ble_myservice_c);
        APP_ERROR_CHECK(err_code);
        NRF_LOG_INFO("Connected to device with My Service.");
        break;

    case BLE_NUS_C_EVT_NUS_TX_EVT:
        ble_nus_chars_received_uart_print(p_ble_myservice_evt->p_data, p_ble_myservice_evt->data_len);
        break;

    case BLE_NUS_C_EVT_DISCONNECTED:
        NRF_LOG_INFO("Disconnected.");
        scan_start();
        break;
    }
}
```



在 `uart_event_handle` 回调中，将收到的串口数据发送到我们自定义服务的`Notify` 属性中。

```c
void uart_event_handle(app_uart_evt_t * p_event)
{
    static uint8_t data_array[BLE_NUS_MAX_DATA_LEN];
    static uint16_t index = 0;
    uint32_t ret_val;

    switch (p_event->evt_type)
    {
    /**@snippet [Handling data from UART] */
    case APP_UART_DATA_READY:
        UNUSED_VARIABLE(app_uart_get(&data_array[index]));
        index++;

        if ((data_array[index - 1] == '\n') ||
                (data_array[index - 1] == '\r') ||
                (index >= (m_ble_nus_max_data_len)))
        {
            NRF_LOG_DEBUG("Ready to send data over BLE NUS");
            NRF_LOG_HEXDUMP_DEBUG(data_array, index);

            do
            {
            /*****************自定义服务**********************/
                ret_val = ble_myservice_c_string_send(&m_ble_myservice_c, data_array, index);
				if ( (ret_val != NRF_ERROR_INVALID_STATE) && (ret_val != NRF_ERROR_RESOURCES) )
                {
                    APP_ERROR_CHECK(ret_val);
                }
                /*******************end********************/
                ret_val = ble_nus_c_string_send(&m_ble_nus_c, data_array, index);
                if ( (ret_val != NRF_ERROR_INVALID_STATE) && (ret_val != NRF_ERROR_RESOURCES) )
                {
                    APP_ERROR_CHECK(ret_val);
                }
            } while (ret_val == NRF_ERROR_RESOURCES);

            index = 0;
        }
        break;
}
```



在发现模块回调处理中添加我们自定义服务的处理函数。

```c
static void db_disc_handler(ble_db_discovery_evt_t * p_evt)
{
    ble_nus_c_on_db_disc_evt(&m_ble_nus_c, p_evt);

    ble_myservice_c_on_db_disc_evt(&m_ble_myservice_c, p_evt);
}
```



自此。我们在中心设备上的处理外围设备的自定义服务的代码就添加好了。

外围设备服务如下，都是 `nus` 服务基础上进行修改的。相当于我们自己新建了另外一个 `nus` 服务。



<div align=center><img src="https://raw.githubusercontent.com/duapple/Code/master/pic/20210127105703.png" alt="Screenshot_2021-01-26-00-03-53-103_no.nordicsemi." style="zoom: 33%;"  width="40%" high="40%"/>

这里我的自定义服务是在 `Nordic 52810` 上跑的。然后将 `Nordic 52840` 作为中心设备来连接这个外围设备。`UART Service` 用工程中原有的。然后`Unknown Service` 采用我们自己的服务处理来进行连接和处理。

这里中心设备为`Nordic Downgle` ，通过串口连接PC， PC串口助手发送数据到 `Downgle`，然后 `Downgle` 会通过 `NUS` 和 `MYSERVICE`  两个服务将串口接收到的数据发送到外围设备对应的两个服务上。

这里DB发现模块，会在建立连接后，按照我们设置的 `uuid` 来依次发现我们定义的服务，并建立连接。

这里，外围设备上对应服务的回数据处理回调（`nus_new_data_handler` 和 `nus_data_handler`）接收到了来自中心设备的两个服务处理的数据。

<div align=center><img src="https://github.com/duapple/Code/blob/master/pic/20210127105733.png?raw=true" alt="Screenshot_2021-01-26-00-03-53-103_no.nordicsemi." style="zoom: 33%;" width="80%" high="80%"/>