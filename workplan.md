okr1. 属性上报





1. 设置 AIOT_MQTTOPT_RECV_HANDLER
    mqtt_handle->recv_handler = (aiot_mqtt_recv_handler_t)data;

2. 在aiot_mqtt_recv函数：
    _core_mqtt_pingresp_handler，
        _core_mqtt_pub_handler，
        _core_mqtt_puback_handler，
        _core_mqtt_subunsuback_handler
    
    执行回调函数
    /* User Ctrl Packet Handler */
    if (mqtt_handle->recv_handler) {
        mqtt_handle->recv_handler((void *)mqtt_handle, &packet, mqtt_handle->userdata);
    }



1. 设置AIOT_MQTTOPT_EVENT_HANDLER
     mqtt_handle->event_handler = (aiot_mqtt_event_handler_t)data;
    
2. 连接/重连， 失连是调用
    if (mqtt_handle->event_handler) {
            mqtt_handle->event_handler((void *)mqtt_handle, &event, mqtt_handle->userdata);
    }

    // 执行mqtt_handle->process_data_list链表
    _core_mqtt_event_notify_process_handler
        mqtt_handle->process_data_list  节点类型  core_mqtt_process_data_node_t
            node->process_data.handler(node->process_data.context, event, core_event);






子设备处理：
1. 映射topic和handler到mqtt_handle的sub_list的sub_node,handler_list
 aiot_subdev_setopt(subdev_handle, AIOT_SUBDEVOPT_MQTT_HANDLE, mqtt_handle);
    _subdev_operate_topic_map(subdev_handle, AIOT_MQTTOPT_APPEND_TOPIC_MAP);    // 映射topic和处理函数结构体到mqtt_sub_list的sub_node的handler_list
        要映射的topic和处理函数的数组结构：g_subdev_topic_map
        函数_core_mqtt_append_topic_map 是添加处理函数到sub_topic的处理函数列表     mqtt_handle->sub_list  节点类型： core_mqtt_sub_node_t,  子节点类型： core_mqtt_sub_handler_node_t
        _core_mqtt_sublist_insert(mqtt_handle, &topic_buff, map->handler, map->userdata);
            //先查找sub_topic是否存在，存在插入sub_node的处理函数列表，不存在新建
            _core_mqtt_handlerlist_insert(mqtt_handle, node, handler, userdata);
                2.  设置
                    aiot_subdev_setopt(subdev_handle, AIOT_SUBDEVOPT_RECV_HANDLER, demo_subdev_recv_handler);
                        subdev_handle->recv_handler = (aiot_subdev_recv_handler_t)data;
                    调用位置
                    _subdev_topo_generic_reply_recv_handler和_subdev_topo_generic_notify_recv_handler中调用
                        subdev_handle->recv_handler(subdev_handle, &recv, subdev_handle->userdata);

2. mqtt_handler中接收pub报文：
    _core_mqtt_pub_handler
        pub来了，寻找是否sub了该topic
        core_list_for_each_entry(sub_node, &mqtt_handle->sub_list, linked_node, core_mqtt_sub_node_t) {
            if (_core_mqtt_topic_compare(sub_node->topic, (uint32_t)(strlen(sub_node->topic)), packet.data.pub.topic,
                                        packet.data.pub.topic_len) == STATE_SUCCESS) {
                _core_mqtt_handlerlist_append(mqtt_handle, &handler_list_copy, &sub_node->handle_list, &sub_found);
            }
        }
        然后，循环该sub_node的handler_list执行：
            core_list_for_each_entry(handler_node, &handler_list_copy,
                             linked_node, core_mqtt_sub_handler_node_t) {
              userdata = (handler_node->userdata == NULL) ? (mqtt_handle->userdata) : (handler_node->userdata);
                 handler_node->handler(mqtt_handle, &packet, userdata);
               }


物模型

1. 设置AIOT_DMOPT_MQTT_HANDLE
    _dm_setup_topic_mapping 成功的话，继续_dm_core_mqtt_operate_process_handler

    从g_dm_recv_topic_mapping中映射 AIOT_MQTTOPT_APPEND_TOPIC_MAP

    添加_dm_core_mqtt_process_handler到CORE_MQTTOPT_APPEND_PROCESS_HANDLER   mqtt_handle->process_data_list  节点类型：core_mqtt_process_data_node_t  子节点类型 core_mqtt_process_data_t


设置recv函数： AIOT_DMOPT_RECV_HANDLER      dm_handle->recv_handler

    _dm_recv_generic_reply_handler, _dm_recv_property_set_handler, _dm_recv_async_service_invoke_handler, _dm_recv_sync_service_invoke_handler, _dm_recv_raw_data_handler, _dm_recv_raw_sync_service_invoke_handler
    中调用：
        dm_handle->recv_handler(dm_handle, &recv, dm_handle->userdata);





