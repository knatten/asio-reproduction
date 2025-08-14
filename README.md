This is a reproduction for a race condition in asio. [asio issue here](https://github.com/chriskohlhoff/asio/issues/1657)

## Bug report:

There seems to be a data race on the per descriptor `conditionally_enabled_mutex` used in `kqueue_reactor`, if the reactor is already running when a new descriptor is registered.

Reproduced on macOS 15.6 on an M4 with Boost Asio 1.88.0 with this slight modification to the [official UPD server example](https://think-async.com/Asio/boost_asio_1_30_2/doc/html/boost_asio/tutorial/tutdaytime6/src.html):

```diff
diff --git a/main.cpp b/main.cpp
index 4fce29a..7e3c19e 100644
--- a/main.cpp
+++ b/main.cpp
@@ -77,8 +77,11 @@ int main()
   try
   {
     boost::asio::io_context io_context;
+    auto work = boost::asio::make_work_guard(io_context);
+    std::thread t([&](){io_context.run();});
     udp_server server(io_context);
-    io_context.run();
+    work.reset();
+    t.join();
   }
   catch (std::exception& e)
   {
```

A full CMake project with the reproduction can be found [here](https://github.com/knatten/asio-reproduction).

To reproduce, run the modified server and send it a udp packet. E.g.
```
echo foo| nc -u -w1 127.0.0.1 13; echo $?
```

## The problem:

The problem lies in the synchronization of the per descriptor `conditionally_enabled_mutex` used in `descriptor_state`. It is allocated by the main thread and read on the worker thread, without proper synchronization. Several mutexes are in play here, but none of them seem to establish a happens-before relationship between the allocation of the `conditionally_enabled_mutex` and its subsequent use in the worker thread.

More details:

When the socket is registered, the following happens in the main thread:
- `kqueue_reactor::register_descriptor` is called
  - `kqueue_reactor::allocate_descriptor_state` is called, which:
    - locks the `kqueue_reactror`'s main mutex `registered_descriptors_mutex_`
    - allocates a new `descriptor_state` object, including the `conditionally_enabled_mutex`
    - unlocks `registered_descriptors_mutex_`
    - (So the allocation of the per descriptor `conditionally_enabled_mutex` is protected by the main `registered_descriptors_mutex_`, which seems good, but doesn't help with this exact problem)
  - locks and unlocks the `conditionally_enabled_mutex` of the new descriptor, establishing the "release" end of a happens-before relationship for anyone who subsequently acquires it (also good but doesn't help us here)

In `kqueue_reactor::run`, the following happens in the worker thread:
- `kqueue_reactor::run` is called, which:
    - locks the `kqueue_reactror`'s main mutex `registered_descriptors_mutex_`
    - unlocks `registered_descriptors_mutex_`
    - calls `kevent`, which returns when there is data on the socket
    - gets the corresponding `descriptor_state` from the `events` member
    - locks the `conditionally_enabled_mutex` of the `descriptor_state` with a `scoped_lock`, which synchronizes with the release in `register_descriptor` (also good, but the data race already happened before this point)

So there's a lot of correct synchronization here, but as far as I can tell, there is no happens-before relationship between the allocation of the `descriptor_state`'s `conditionally_enabled_mutex` itself and the subsequent locking of it in the worker thread.

Technically, the new descriptor state could be allocated after the worker thread has already locked and unlocked the `registered_descriptors_mutex`, leaving no more synchronization points before the new descriptor state is used by the worker thread.

This is unlikely to cause a problem in practice, but is technically a data race and undefined behavior.

## Thread sanitizer:
The problem has been confirmed with thread sanitizer, which reports the following:

```
Data race (pid=46806)
Read of size 1 at 0x000106d0267c by thread T1:
  at 0x000102544364 boost::asio::detail::conditionally_enabled_mutex::scoped_lock::scoped_lock(boost::asio::detail::conditionally_enabled_mutex&) (conditionally_enabled_mutex.hpp:53)
  at 0x000102543c70 boost::asio::detail::conditionally_enabled_mutex::scoped_lock::scoped_lock(boost::asio::detail::conditionally_enabled_mutex&) (conditionally_enabled_mutex.hpp:52)
  at 0x000102540d38 boost::asio::detail::kqueue_reactor::run(long, boost::asio::detail::op_queue<boost::asio::detail::scheduler_operation>&) (kqueue_reactor.ipp:494)
  at 0x000102540bb4 boost::asio::detail::kqueue_reactor::run(long, boost::asio::detail::op_queue<boost::asio::detail::scheduler_operation>&) (kqueue_reactor.ipp:445)
  at 0x000102547984 boost::asio::detail::scheduler::do_run_one(boost::asio::detail::conditionally_enabled_mutex::scoped_lock&, boost::asio::detail::scheduler_thread_info&, boost::system::error_code const&) (scheduler.ipp:475)
  at 0x00010254744c boost::asio::detail::scheduler::run(boost::system::error_code&) (scheduler.ipp:208)
  at 0x00010256afa0 boost::asio::io_context::run() (io_context.ipp:71)
  at 0x00010256af24 main::$_0::operator()() const (main.cpp:81)
  at 0x00010256ae74 decltype(std::declval<main::$_0>()()) std::__1::__invoke[abi:ne190102]<main::$_0>(main::$_0&&) (invoke.h:149)
  at 0x00010256ada0 void std::__1::__thread_execute[abi:ne190102]<std::__1::unique_ptr<std::__1::__thread_struct, std::__1::default_delete<std::__1::__thread_struct>>, main::$_0>(std::__1::tuple<std::__1::unique_ptr<std::__1::__thread_struct, std::__1::default_delete<std::__1::__thread_struct>>, main::$_0>&, std::__1::__tuple_indices<>) (thread.h:198)
  at 0x00010256a154 void* std::__1::__thread_proxy[abi:ne190102]<std::__1::tuple<std::__1::unique_ptr<std::__1::__thread_struct, std::__1::default_delete<std::__1::__thread_struct>>, main::$_0>>(void*) (thread.h:207)
Previous write of size 1 at 0x000106d0267c by main thread (mutexes: write M0):
  at 0x0001025426a4 boost::asio::detail::conditionally_enabled_mutex::conditionally_enabled_mutex(bool, int) (conditionally_enabled_mutex.hpp:118)
  at 0x00010253fe58 boost::asio::detail::conditionally_enabled_mutex::conditionally_enabled_mutex(bool, int) (conditionally_enabled_mutex.hpp:119)
  at 0x000102542f30 boost::asio::detail::kqueue_reactor::descriptor_state::descriptor_state(bool, int) (kqueue_reactor.hpp:70)
  at 0x000102542ea8 boost::asio::detail::kqueue_reactor::descriptor_state::descriptor_state(bool, int) (kqueue_reactor.hpp:70)
  at 0x000102542dbc boost::asio::detail::kqueue_reactor::descriptor_state* boost::asio::detail::object_pool_access::create<boost::asio::detail::kqueue_reactor::descriptor_state, bool, int>(bool, int) (object_pool.hpp:42)
  at 0x00010255a7a8 boost::asio::detail::kqueue_reactor::descriptor_state* boost::asio::detail::object_pool<boost::asio::detail::kqueue_reactor::descriptor_state>::alloc<bool, int>(bool, int) (object_pool.hpp:106)
  at 0x00010255a678 boost::asio::detail::kqueue_reactor::allocate_descriptor_state() (kqueue_reactor.ipp:573)
  at 0x00010255a274 boost::asio::detail::kqueue_reactor::register_descriptor(int, boost::asio::detail::kqueue_reactor::descriptor_state*&) (kqueue_reactor.ipp:152)
  at 0x000102559d0c boost::asio::detail::reactive_socket_service_base::do_open(boost::asio::detail::reactive_socket_service_base::base_implementation_type&, int, int, int, boost::system::error_code&) (reactive_socket_service_base.ipp:188)
  at 0x00010254b214 boost::asio::detail::reactive_socket_service<boost::asio::ip::udp>::open(boost::asio::detail::reactive_socket_service<boost::asio::ip::udp>::implementation_type&, boost::asio::ip::udp const&, boost::system::error_code&) (reactive_socket_service.hpp:129)
  at 0x00010254af18 boost::asio::basic_socket<boost::asio::ip::udp, boost::asio::any_io_executor>::basic_socket<boost::asio::io_context>(boost::asio::io_context&, boost::asio::ip::basic_endpoint<boost::asio::ip::udp> const&, boost::asio::constraint<is_convertible<boost::asio::io_context&, boost::asio::execution_context&>::value, int>::type) (basic_socket.hpp:237)
  at 0x00010254ae34 boost::asio::basic_datagram_socket<boost::asio::ip::udp, boost::asio::any_io_executor>::basic_datagram_socket<boost::asio::io_context>(boost::asio::io_context&, boost::asio::ip::basic_endpoint<boost::asio::ip::udp> const&, boost::asio::constraint<is_convertible<boost::asio::io_context&, boost::asio::execution_context&>::value, int>::type) (basic_datagram_socket.hpp:205)
  at 0x00010254a780 boost::asio::basic_datagram_socket<boost::asio::ip::udp, boost::asio::any_io_executor>::basic_datagram_socket<boost::asio::io_context>(boost::asio::io_context&, boost::asio::ip::basic_endpoint<boost::asio::ip::udp> const&, boost::asio::constraint<is_convertible<boost::asio::io_context&, boost::asio::execution_context&>::value, int>::type) (basic_datagram_socket.hpp:206)
  at 0x00010254a5f8 udp_server::udp_server(boost::asio::io_context&) (main.cpp:32)
  at 0x0001025359d8 udp_server::udp_server(boost::asio::io_context&) (main.cpp:33)
  at 0x000102535664 main (main.cpp:82)
Location is heap block of size 160 at 0x000106d02620 allocated by main thread:
  at 0x000102ba66b4 operator new(unsigned long)
  at 0x000102542da0 boost::asio::detail::kqueue_reactor::descriptor_state* boost::asio::detail::object_pool_access::create<boost::asio::detail::kqueue_reactor::descriptor_state, bool, int>(bool, int) (object_pool.hpp:42)
  at 0x00010255a7a8 boost::asio::detail::kqueue_reactor::descriptor_state* boost::asio::detail::object_pool<boost::asio::detail::kqueue_reactor::descriptor_state>::alloc<bool, int>(bool, int) (object_pool.hpp:106)
  at 0x00010255a678 boost::asio::detail::kqueue_reactor::allocate_descriptor_state() (kqueue_reactor.ipp:573)
  at 0x00010255a274 boost::asio::detail::kqueue_reactor::register_descriptor(int, boost::asio::detail::kqueue_reactor::descriptor_state*&) (kqueue_reactor.ipp:152)
  at 0x000102559d0c boost::asio::detail::reactive_socket_service_base::do_open(boost::asio::detail::reactive_socket_service_base::base_implementation_type&, int, int, int, boost::system::error_code&) (reactive_socket_service_base.ipp:188)
  at 0x00010254b214 boost::asio::detail::reactive_socket_service<boost::asio::ip::udp>::open(boost::asio::detail::reactive_socket_service<boost::asio::ip::udp>::implementation_type&, boost::asio::ip::udp const&, boost::system::error_code&) (reactive_socket_service.hpp:129)
  at 0x00010254af18 boost::asio::basic_socket<boost::asio::ip::udp, boost::asio::any_io_executor>::basic_socket<boost::asio::io_context>(boost::asio::io_context&, boost::asio::ip::basic_endpoint<boost::asio::ip::udp> const&, boost::asio::constraint<is_convertible<boost::asio::io_context&, boost::asio::execution_context&>::value, int>::type) (basic_socket.hpp:237)
  at 0x00010254ae34 boost::asio::basic_datagram_socket<boost::asio::ip::udp, boost::asio::any_io_executor>::basic_datagram_socket<boost::asio::io_context>(boost::asio::io_context&, boost::asio::ip::basic_endpoint<boost::asio::ip::udp> const&, boost::asio::constraint<is_convertible<boost::asio::io_context&, boost::asio::execution_context&>::value, int>::type) (basic_datagram_socket.hpp:205)
  at 0x00010254a780 boost::asio::basic_datagram_socket<boost::asio::ip::udp, boost::asio::any_io_executor>::basic_datagram_socket<boost::asio::io_context>(boost::asio::io_context&, boost::asio::ip::basic_endpoint<boost::asio::ip::udp> const&, boost::asio::constraint<is_convertible<boost::asio::io_context&, boost::asio::execution_context&>::value, int>::type) (basic_datagram_socket.hpp:206)
  at 0x00010254a5f8 udp_server::udp_server(boost::asio::io_context&) (main.cpp:32)
  at 0x0001025359d8 udp_server::udp_server(boost::asio::io_context&) (main.cpp:33)
  at 0x000102535664 main (main.cpp:82)
Mutex M0 (0x000106903df0) created at:
  at 0x000102b50598 pthread_mutex_init
  at 0x00010253bcb8 boost::asio::detail::posix_mutex::posix_mutex() (posix_mutex.ipp:34)
  at 0x00010253bc50 boost::asio::detail::posix_mutex::posix_mutex() (posix_mutex.ipp:33)
  at 0x00010254266c boost::asio::detail::conditionally_enabled_mutex::conditionally_enabled_mutex(bool, int) (conditionally_enabled_mutex.hpp:116)
  at 0x00010253fe58 boost::asio::detail::conditionally_enabled_mutex::conditionally_enabled_mutex(bool, int) (conditionally_enabled_mutex.hpp:119)
  at 0x00010253f7dc boost::asio::detail::kqueue_reactor::kqueue_reactor(boost::asio::execution_context&) (kqueue_reactor.ipp:59)
  at 0x00010253f51c boost::asio::detail::kqueue_reactor::kqueue_reactor(boost::asio::execution_context&) (kqueue_reactor.ipp:63)
  at 0x00010253f154 boost::asio::execution_context::service* boost::asio::detail::service_registry::create<boost::asio::detail::kqueue_reactor, boost::asio::execution_context>(void*) (service_registry.hpp:86)
  at 0x00010253f2ac boost::asio::detail::service_registry::do_use_service(boost::asio::execution_context::service::key const&, boost::asio::execution_context::service* (*)(void*), void*) (service_registry.ipp:132)
  at 0x00010253f09c boost::asio::detail::kqueue_reactor& boost::asio::detail::service_registry::use_service<boost::asio::detail::kqueue_reactor>() (service_registry.hpp:30)
  at 0x00010253efec boost::asio::detail::kqueue_reactor& boost::asio::use_service<boost::asio::detail::kqueue_reactor>(boost::asio::execution_context&) (execution_context.hpp:35)
  at 0x00010254bc10 boost::asio::detail::reactive_socket_service_base::reactive_socket_service_base(boost::asio::execution_context&) (reactive_socket_service_base.ipp:34)
  at 0x00010254bab0 boost::asio::detail::reactive_socket_service<boost::asio::ip::udp>::reactive_socket_service(boost::asio::execution_context&) (reactive_socket_service.hpp:81)
  at 0x00010254ba30 boost::asio::detail::reactive_socket_service<boost::asio::ip::udp>::reactive_socket_service(boost::asio::execution_context&) (reactive_socket_service.hpp:82)
  at 0x00010254b90c boost::asio::execution_context::service* boost::asio::detail::service_registry::create<boost::asio::detail::reactive_socket_service<boost::asio::ip::udp>, boost::asio::io_context>(void*) (service_registry.hpp:86)
  at 0x00010253f2ac boost::asio::detail::service_registry::do_use_service(boost::asio::execution_context::service::key const&, boost::asio::execution_context::service* (*)(void*), void*) (service_registry.ipp:132)
  at 0x00010254b854 boost::asio::detail::reactive_socket_service<boost::asio::ip::udp>& boost::asio::detail::service_registry::use_service<boost::asio::detail::reactive_socket_service<boost::asio::ip::udp>>(boost::asio::io_context&) (service_registry.hpp:39)
  at 0x00010254b57c boost::asio::detail::reactive_socket_service<boost::asio::ip::udp>& boost::asio::use_service<boost::asio::detail::reactive_socket_service<boost::asio::ip::udp>>(boost::asio::io_context&) (io_context.hpp:41)
  at 0x00010254b43c boost::asio::detail::io_object_impl<boost::asio::detail::reactive_socket_service<boost::asio::ip::udp>, boost::asio::any_io_executor>::io_object_impl<boost::asio::io_context>(int, int, boost::asio::io_context&) (io_object_impl.hpp:58)
  at 0x00010254b088 boost::asio::detail::io_object_impl<boost::asio::detail::reactive_socket_service<boost::asio::ip::udp>, boost::asio::any_io_executor>::io_object_impl<boost::asio::io_context>(int, int, boost::asio::io_context&) (io_object_impl.hpp:60)
  at 0x00010254aec4 boost::asio::basic_socket<boost::asio::ip::udp, boost::asio::any_io_executor>::basic_socket<boost::asio::io_context>(boost::asio::io_context&, boost::asio::ip::basic_endpoint<boost::asio::ip::udp> const&, boost::asio::constraint<is_convertible<boost::asio::io_context&, boost::asio::execution_context&>::value, int>::type) (basic_socket.hpp:233)
  at 0x00010254ae34 boost::asio::basic_datagram_socket<boost::asio::ip::udp, boost::asio::any_io_executor>::basic_datagram_socket<boost::asio::io_context>(boost::asio::io_context&, boost::asio::ip::basic_endpoint<boost::asio::ip::udp> const&, boost::asio::constraint<is_convertible<boost::asio::io_context&, boost::asio::execution_context&>::value, int>::type) (basic_datagram_socket.hpp:205)
  at 0x00010254a780 boost::asio::basic_datagram_socket<boost::asio::ip::udp, boost::asio::any_io_executor>::basic_datagram_socket<boost::asio::io_context>(boost::asio::io_context&, boost::asio::ip::basic_endpoint<boost::asio::ip::udp> const&, boost::asio::constraint<is_convertible<boost::asio::io_context&, boost::asio::execution_context&>::value, int>::type) (basic_datagram_socket.hpp:206)
  at 0x00010254a5f8 udp_server::udp_server(boost::asio::io_context&) (main.cpp:32)
  at 0x0001025359d8 udp_server::udp_server(boost::asio::io_context&) (main.cpp:33)
  at 0x000102535664 main (main.cpp:82)
Thread T1 (tid=80109680, running) created by main thread at:
  at 0x000102b4eba8 pthread_create
  at 0x00010256a0bc std::__1::__libcpp_thread_create[abi:ne190102](_opaque_pthread_t**, void* (*)(void*), void*) (pthread.h:182)
  at 0x000102569e68 std::__1::thread::thread<main::$_0, 0>(main::$_0&&) (thread.h:217)
  at 0x00010253596c std::__1::thread::thread<main::$_0, 0>(main::$_0&&) (thread.h:212)
  at 0x000102535654 main (main.cpp:81)
```

Note: I'm aware that asio uses fenced locks, which are not supported by thread sanitizer, possibly leading to false positives. However, I've placed breakpoints on the `std_fenced_block` constructors and confirmed that no such fence is created before thread sanitizer reports the issue.