add_executable(orderbook_tests init_test.cpp)

target_link_libraries(orderbook_tests
  PRIVATE
  orderbook
  Catch2::Catch2WithMain
)
include(Catch)
catch_discover_tests(orderbook_tests)
