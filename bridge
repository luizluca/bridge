#!/usr/bin/ruby
#
#  Copyright (C) 2010 by Luiz Angelo Daros de Luca
#    luizluca@gmail.com
#                                               
#    This program is free software; you can redistribute it and/or modify
#    it under the terms of the GNU General Public License as published by
#    the Free Software Foundation; either version 2 of the License, or
#    any later version.  
#                                                                     
#    This program is distributed in the hope that it will be useful,  
#    but WITHOUT ANY WARRANTY; without even the implied warranty of  
#    MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the  
#    GNU General Public License for more details.                  
#  
#    Created on 2010-08-10
#
# Changelog:
#
# * 2010-09-01
# - Add TCP keepalive flags for local socket (thanks Jan Adámek)
# - Increased local buffer to 640k
# - Fixed sync problem with stdin/out when using as ProxyCommand inside ssh config
# - Reconnection on client-webrick disconnection
# - Fixed proxy absence 
# - Specified webrick timeout (10m)
# - Treated more signals
# - Divide connection (open, read, write)
#
#################
#
# Port Forward client/server over HTTP 
#
#################

case ARGV.size
	when 2
		server=true
        location=ARGV[1]
 	when 4
		server=false
        url=ARGV[1]
		remHost=ARGV[2]
		remPort=ARGV[3].to_i
	else
	$stderr.puts <<EOF
Use as server: 
#$0 localPort bridgeLocation

Use as client:
#$0 localPort|STDIN bridgeURL remoteAddress remotePort 

Ex:
bridge$ #$0 8080 /bridge

client$ #$0 8022 http://mybridge:8080/bridge remoteServer 22
client$ ssh localhost -p 8022
EOF
    exit
end

localPort=ARGV[0]

require 'socket' 
require 'thread'
Thread.abort_on_exception = true

MAXBUF=1024*640 # 640k
if server
	require 'webrick'

	class BridgeServlet < WEBrick::HTTPServlet::AbstractServlet
        @@connections = {}

        def getSocket(req,res)
            cid = req.path.split("/").last
		    s = @@connections[cid]
            if not s
                $stderr.puts "Connection is not open for #{cid}=#{s}"
                res.status=404
                return nil
            end
            s
        end

        def closeSocket(req,res)
            cid = req.path.split("/").last
        	return if not s=@@connections[cid]
            s.close if not s.closed?
            @@connections.delete(cid)
        end

        def do_POST(req, res)
            cid = req.path.split("/").last
			(remHost,remPort)=req.body.split(":")
            remPort=remPort.to_i
			$stderr.puts "Opening connection to #{remHost}:#{remPort}..."
            begin
			    s=TCPSocket.open(remHost,remPort)
                s.setsockopt(Socket::SOL_SOCKET, Socket::SO_KEEPALIVE, true)
                @@connections[cid]=s
			    $stderr.puts "Connected to #{remHost}:#{remPort}"
            rescue 
			    res['Content-Type'] = "text/plain"
                res.body=$!.message
				$stderr.puts "Connection failed: #$!"
                res.status=503 # Service Unavaiable
            end
        end

		def do_PUT(req, res)
            cid = req.path.split("/").last
            return if not s=getSocket(req,res)
            begin
			    s.print req.body
            rescue Errno::EPIPE
                $stderr.puts "Connection closed in remote destination (PUT)"
                closeSocket(req, res)
                res.status=410 # GONE
            end
		end

		def do_GET(req, res)
            cid = req.path.split("/").last
            return if not s=getSocket(req,res)
            begin
			    res.body = s.readpartial(MAXBUF)
            rescue EOFError, Errno::EPIPE, IOError
                if s.closed?
                    $stderr.puts "Connection closed by remote destination (GET)"
                else
                    $stderr.puts "Connection closed (GET) #{$!}"
                end
                closeSocket(req, res)
                res.status=410 # GONE
            rescue 
                $stderr.puts $!.class
            end            
		end

        def do_DELETE(req, res)
            $stderr.puts "Connection closed in client. Trying to close it in remote destination"
            closeSocket(req, res)
        end

	end

	s = WEBrick::HTTPServer.new(
		:Port            => localPort.to_i,
        :RequestTimeout  => 600
	)
	s.mount(location, BridgeServlet)
	trap("INT"){ s.shutdown }
	s.start
else
	require "net/http"
	require 'uri'

    proxy=[nil,nil]
    if ENV["http_proxy"] and not ENV["http_proxy"].empty?
	    proxy=ENV["http_proxy"].gsub(/\/*/,"").split(":")[1..2]   
        $stderr.puts "Using proxy: #{proxy[0]}:#{proxy[1]}"
    end

    case localPort 
    when "STDIN","-"
        sIn=$stdin
       	sOut=$stdout
        # Keeps this unbuffered
        sIn.sync=true
        sOut.sync=true
    else
        $stderr.puts "Opening local port #{localPort}"
       	server = TCPServer.open(localPort.to_i)
        $stderr.puts "Waiting for local connection to port #{localPort}"
      	s = server.accept 
        sIn=sOut=s
        # Keep connection alive
        s.setsockopt(Socket::SOL_SOCKET, Socket::SO_KEEPALIVE, true)
    end

   	url = URI.parse(url)
    #unique Connection IDentifier
    cid=`uuidgen`.chomp
    connected=false

    # Open the connection
    begin
        begin
            Net::HTTP::Proxy(proxy[0],proxy[1]).start(url.host, url.port) {|http|
                $stderr.puts "Opening connection over HTTP bridge for #{remHost}:#{remPort} (#{cid})"
                error=nil
                res = http.post("#{url.path}/#{cid}","#{remHost}:#{remPort}") {|res| error=res }            
                if res.kind_of? Net::HTTPServiceUnavailable
                    $stderr.puts "The bridge failed to connect to the remote location: #{error}"
                    connected=false
                    break
                else
                    connected=true
                end
            }
        rescue Errno::EPIPE
            $stderr.puts "Connection to bridge closed"
            retry
        end

        # Not connected, nothing more to do
        exit 1 if not connected

        # Launch Local write/Remote read loop
        Thread.new {
            $stderr.puts "Local write/Remote read loop started"
            begin
                Net::HTTP::Proxy(proxy[0],proxy[1]).start(url.host, url.port) {|http|
                    http.read_timeout = 2>>64

                    while connected
                        res = http.get("#{url.path}/#{cid}") {|res|
                            sOut.print res 
                        }
                        if res.kind_of? Net::HTTPGone
                            $stderr.puts "Connection closed in remote location (lw/rr)"
                            connected=false
                            sIn.close if not sIn.closed?
                            break
                        elsif res.kind_of? Net::HTTPNotFound
                            $stderr.puts "Connection not oppened by bridge"
                            connected=false
                            break
                        end
                    end
                } 
            rescue Errno::EPIPE
                sIn.close if not sIn.closed?
            rescue EOFError
                # retry if local connection is still open
                retry if not sIn.closed?
                $stderr.puts "Connection to bridge closed (lr/rw)"
            rescue Errno::ECONNREFUSED
                $stderr.puts "Connection to bridge failed (lr/rw)"
                sIn.close if not sIn.closed?
            rescue Timeout::Error
                $stderr.puts "Timeout (lr/rw)"
                retry if not sIn.closed?
            rescue Net::HTTPServerError
                raise res.message
            end
        }	

        # If CTRL+C, SIGHUP or SIGTERM, close sIn (and itself)
        trap("INT"){ sIn.close } 
        trap("SIGHUP"){ sIn.close } 
        trap("SIGTERM"){ sIn.close } 

        # Launch "Local read/Remote write loop started"
        $stderr.puts "Local read/Remote write loop started"
        # Keep buffer between retries
        buf=nil
        begin
            Net::HTTP::Proxy(proxy[0],proxy[1]).start(url.host, url.port) {|http|
                http.read_timeout = 2>>64

                while buf or (connected and buf=sIn.readpartial(MAXBUF))
                    res = http.put("#{url.path}/#{cid}",buf)
                    if res.kind_of? Net::HTTPGone
                        $stderr.puts "Connection closed in remote location (lr/rw)"
                        sIn.close if not sIn.closed?
                        connected=false
                        break 
                    end
                    # Buffer sent, clear it
                    buf=nil
                end
                if connected
                    connected=false
                    $stderr.puts "Closing bridge connection to remote location"
                    http.delete("#{url.path}/#{cid}")
                end
            }
        rescue Net::HTTPServerError
            raise "Server said: '#{res.message}'"
        rescue EOFError
            $stderr.puts "Local connection closed (lr/rw)"
            connected=false
        rescue Errno::EPIPE, IOError
            # retry if local connection is still open
            retry if not sIn.closed?
            $stderr.puts "Connection to bridge closed (lr/rw)"
        end
    rescue Errno::ECONNREFUSED
        $stderr.puts "Connection to bridge failed"
    end
end

$stderr.puts "Program Finished!"
