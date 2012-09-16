# `__repmgr_start()`

This function is the entry point of replication manager.

If there is no selector thead exists, spawn one.

		if ((ret = __repmgr_start_selector(env)) != 0)

This thread is used to do the main io work, like reading or writing
data from/to each connection, invoking the corresponding callback
function of each connection, pushing messages received from other
sites to the message queue.

		if ((ret =
		    __repmgr_start_msg_threads(env, (u_int)nthreads)) != 0)
			goto err;


Then serveral message processing thread will be spawned. The real work
is done in `message_loop()`. This function keeps popping from the
message queue and process the message.

# `process_message()`

This method first calls the `__rep_process_message_int()` to perform
te internal steps to process a incoming message, such as checking the
version of the replication message. If the message version is an old
version the local site supports, convert it. Otherwise there will be
an error. So this has to be considered when upgrading berkeley db.

