---
title: 'CS144: Introduction to Computer Networking Lab 6 & lab 7'
tags:
  - tag_name
categories:
  - category_name
date: 2022-11-30 20:05:36
---


## Lab 6

### add_route

To add a router, we need to store `route_prefix`, `prefix_length`, `next_hop` and `interface_num`.

So I design the following data structure:

```c++
class Router {
    ...
    //! The router information
    struct RouterInformation {
        uint32_t route_prefix;
        uint8_t prefix_length;
        std::optional<Address> next_hop;
        size_t interface_num;
    };

    //! The router table
    std::vector<RouterInformation> _router_table{};
}
```

Now it is easy to implement `add_route`:

```c++
void Router::add_route(const uint32_t route_prefix,
                       const uint8_t prefix_length,
                       const optional<Address> next_hop,
                       const size_t interface_num) {
    cerr << "DEBUG: adding route " << Address::from_ipv4_numeric(route_prefix).ip() << "/" << int(prefix_length)
         << " => " << (next_hop.has_value() ? next_hop->ip() : "(direct)") << " on interface " << interface_num << "\n";

    RouterInformation record{};
    record.route_prefix = route_prefix;
    record.prefix_length = prefix_length;
    record.next_hop = next_hop;
    record.interface_num = interface_num;
    _router_table.push_back(record);
}
```

### route_one_datagram

The process is easy, we just do mask operation. The most tricky part is how to calculate the mask. For example, for `_prefix_length = 16`, we should produce `0x0000ffff`. where we could represent with `uint32_t mask = 0xffffffff >> _prefix_length`. But the trick thing is when `_prefix_length` equals to 32. We should handle this corner case.

```c++

void Router::route_one_datagram(InternetDatagram &dgram) {

    int max_length = -1;
    optional<Address> next_hop {};
    int interface_num = -1;

    // We just iterate the record and fined the longest-prefix-match
    for (auto&& record : _router_table) {
        uint32_t mask = record.prefix_length == 0 ? 0 : numeric_limits<int>::min() >> (record.prefix_length - 1);
        if ((dgram.header().dst & mask) == record.route_prefix && max_length < record.prefix_length) {
            max_length = record.prefix_length;
            next_hop = record.next_hop;
            interface_num = record.interface_num;
        }
    }

    if (max_length == -1 || dgram.header().ttl == 0 || --dgram.header().ttl == 0) {
        return;
    }

    if (next_hop.has_value()) {
        _interfaces[interface_num].send_datagram(dgram, next_hop.value());
    } else {
        _interfaces[interface_num].send_datagram(dgram, Address::from_ipv4_numeric(dgram.header().dst));
    }
}
```

## Lab 7

There is no content in the Lab 7. At last, I wanna appreciate the CS144 team which provides the public this wonderful series of labs.

> What I cannot build, I do not understand.
