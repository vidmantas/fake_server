# FakeServer
[![Build Status](https://travis-ci.org/bernardolins/fake_server.svg?branch=master)](https://travis-ci.org/bernardolins/fake_server)
[![Coverage Status](https://coveralls.io/repos/github/bernardolins/fake_server/badge.svg?branch=master)](https://coveralls.io/github/bernardolins/fake_server?branch=master)

FakeServer is a simple HTTP server used to simulate external services instability on your tests. When you create the server, you provides a list of status, and the requests will be responded with those status, in order of arrival. If there are no more status, the server will respond always 200.

## Basic Usage

```elixir
defmodule UserTest do
  use ExUnit.Case

  setup_all do
    # create some status that your external server could respond with
    # you just need to do it once for you entire test suite.
    FakeServer.Status.create(:status200, %{response_code: 200, response_body: ~s<"username": "mr_user">})
    FakeServer.Status.create(:status500, %{response_code: 500, response_body: ~s<"error": "internal server error">})
    FakeServer.Status.create(:status403, %{response_code: 403, response_body: ~s<"error": "forbidden">})
    :ok
  end
  
  test "#get returns user if the external server responds 200" do
    # start a fake server with a list of status
    {:ok, address} = FakeServer.Server.run(:external_server, :status200)
    # tell your application to access the server
    System.put_env(:external_server_url, address)
    assert User.get == %{username: "mr_user"}
    # stop the server
    FakeServer.Server.stop(:external_server)
  end
  
  test "#get retry up to 3 times when external server responds with 500" do
    {:ok, address} = FakeServer.Server.run(:external_server, [:status500, :status500, :status500, :status200])
    System.put_env(:external_server_url, address)
    # user will be get after 3 retrys
    assert User.get == %{username: "mr_user"}
    FakeServer.Server.stop(:external_server)
  end
  
  test "#get returns timeout after 3 retrys" do
    {:ok, address} = FakeServer.Server.run(:external_server, [:status500, :status500, :status500, :status500])
    System.put_env(:external_server_url, address)
    assert User.get == %{error: "timeout", code: 408}
    FakeServer.Server.stop(:external_server)
  end
end
