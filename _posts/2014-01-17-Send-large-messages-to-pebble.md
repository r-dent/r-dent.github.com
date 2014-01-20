---
layout: post
category : pebble
tagline: "on iOS"
tags : [pebble, smartwatch, AppMessage, AppSync, tutorial]
excerpt: As i tried to build a communication between my iOS and Pebble App like in this example, i ran into issues with the message size really fast.
---
{% include JB/setup %}

As i tried to build a communication between my iOS and Pebble App like in [this example](https://github.com/pebble/pebble-sdk-examples/blob/master/todolist-demo/todo_list/src/todo_list.c), i ran into issues with the message size really fast.
All the examples i found sent messages like this:

    @{@(0): @"a short string",
      @(2): @"another string, but not much longer"}
      
So long as the message does not exceed this general size, i had no problems getting it to the phone. But the information i wanted to transport was a bit larger. To manage it, i built an enum that defines all the keys for my properties. The same enum i defined on the watch side so i could identify the values in the message. With this structure, i wanted to send a larger dictionary to the Watch. My first approach looked like this:

    enum {
      ENTITY_TITLE,
      ENTITY_PROPERTY_1,
      ENTITY_PROPERTY_2,
      ENTITY_PROPERTY_3,
      ENTITY_PROPERTY_4,
      ENTITY_PROPERTY_5
    }
    
    - (void)sendObjectToWatch:(MyObject*)object {
        NSDictionary *message = @{@(ENTITY_TITLE): object.title,
                                  @(ENTITY_PROPERTY_1): object.property1,
                                  @(ENTITY_PROPERTY_2): object.property2,
                                  @(ENTITY_PROPERTY_3): object.property3,
                                  @(ENTITY_PROPERTY_4): object.property4,
                                  @(ENTITY_PROPERTY_5): object.property5};
                                  
        [_watch appMessagesPushUpdate:message onSent:^(PBWatch *watch, NSDictionary *update, NSError *error) {
            if (error != nil)
                NSLog(@"Error sending Pebble message: %@", error);
        }];
    }
    
The title and properties of the object are NSStrings between 10 and 50 characters in lenght. But i got an error every time i tried sending the message. Something like "the watchapp rejected the message" or "the watchapp did not acknowledge the message in time". On the watch side i logged like this:

    void sync_error_callback(DictionaryResult dict_error, AppMessageResult app_message_error, void *context) {
      APP_LOG(APP_LOG_LEVEL_DEBUG, "sync error: %i", app_message_error);
    }
    
This brought me an error code 128. Possible values are described [here](https://developer.getpebble.com/2/api-reference/group___app_message.html#ga695a78c926b20edbb14d7faf5a78c29e). But lacking a mapping between int and enum i could just guess that it was a buffer overflow.
After banging my head against the wall a while, not beleving that these messages are that limited in size, i finally got it and tried a different approach.

##Splitting up the Message

This time i tried to send each property at once.

    - (void)sendObjectToWatch:(MyObject*)object {
        NSDictionary *message = @{@(ENTITY_TITLE): object.title,
                                  @(ENTITY_PROPERTY_1): object.property1,
                                  @(ENTITY_PROPERTY_2): object.property2,
                                  @(ENTITY_PROPERTY_3): object.property3,
                                  @(ENTITY_PROPERTY_4): object.property4,
                                  @(ENTITY_PROPERTY_5): object.property5};
                                  
        for (NSNumber *key in message) {
          NSDictionary *part = @{key: message[key]}
          [_watch appMessagesPushUpdate:part onSent:^(PBWatch *watch, NSDictionary *update, NSError *error) {
            if (error != nil)
              NSLog(@"Error sending Pebble message: %@", error);
          }];
        }
    }
    
This way, a good part of the messages arrived. The other part fired an error 64 on the watch side. My thought was that i might send them too fast.

##Final approach: Waiting for the ACK

So finally i added a queue to that approach.

    @property (nonatomic, strong) NSMutableArray *messageBuffer;
    @property (nonatomic, readonly) BOOL isSending;

    - (void)sendObjectToWatch:(MyObject*)object {
        ...
        for (NSNumber *key in message) {
          NSDictionary *part = @{key: message[key]}
          [self sendMessage:part];
        }
    }
    
    - (void)sendMessage:(NSDictionary*)message {
        if (_isSending)
            [self.messageBuffer addObject:message];
        else {
            _isSending = YES;
            
            [_watch appMessagesPushUpdate:message onSent:^(PBWatch *watch, NSDictionary *update, NSError *error) {
                [_messageBuffer removeObject:message];
                
                if (error != nil)
                    XLog(@"Error sending Pebble message: %@", error);
                
                _isSending = NO;
                
                if (_messageBuffer.count)
                    [self sendMessage:_messageBuffer[0]];
            }];
        }
    }
    
This now worked for me. The messages are were all transferred to the watch. Yippie!
A drawback of this method is that i now see the ui patially updating as the messages are dropping in. But that can be solved by a buffer on the watch side, i guess.

##On the Watch

For now i just write the values direcly on the screen. I use [AppSync](https://developer.getpebble.com/2/api-reference/group___app_sync.html)(Usage covered in [this article](https://developer.getpebble.com/2/guides/app-phone-communication.html)) on the watch to manage message receiving and ui updating.

    void sync_tuple_changed_callback(const uint32_t key, const Tuple* new_tuple, const Tuple* old_tuple, void* context) {
      APP_LOG(APP_LOG_LEVEL_DEBUG, "Received message.");
      write_in_line(key, new_tuple->value->cstring);
    }
    
    void sync_error_callback(DictionaryResult dict_error, AppMessageResult app_message_error, void *context) {
      APP_LOG(APP_LOG_LEVEL_DEBUG, "sync error: %i", app_message_error);
    }
    
    static void init_app_sync() {
      int inbound_size = app_message_inbox_size_maximum();
      int outbound_size = app_message_outbox_size_maximum();
      app_message_open(inbound_size, outbound_size);
    
      Tuplet initial_data[] = {
        TupletCString(ENTITY_TITLE, ""),
        TupletCString(ENTITY_PROPERTY_1, ""),
        TupletCString(ENTITY_PROPERTY_2, ""),
        TupletCString(ENTITY_PROPERTY_3, ""),
        TupletCString(ENTITY_PROPERTY_4, ""),
        TupletCString(ENTITY_PROPERTY_5, "")
      };
    
      app_sync_init(&sync, sync_buffer, sizeof(sync_buffer), initial_data, ARRAY_LENGTH(initial_data), sync_tuple_changed_callback, sync_error_callback, NULL);
    }

##Conclusion

I´m happy with my approach for now. I am now able to send a "large" object to the watch and handle it there. However, the limitations of the watch app are hard to get accustomed with when coming from iOS. 
