project(
  'wolfssl',
  'c',
  version: '5.8.4',
  default_options: [
    'c_std=c99',
    'warning_level=2',
    'b_lto=true',
    'b_ndebug=true'
  ],
  meson_version: '>=1.1.0'
)

cc = meson.get_compiler('c')
c_args = []
c_defs = []
deps = []
src = []
opts = []

if cc.get_id() == 'msvc'
    c_args += [
        '/W3',
        '/wd5045' # Compiler will insert Spectre mitigation for memory load if /Qspectre switch specified
    ]
else
    c_args += [ '-Wall', '-Wextra', '-Wno-unused' ]
endif

_headers = [
    'limits.h',
    'stddef.h',
    'time.h',
    'errno.h',
    'inttypes.h',
    'memory.h',
    'stdint.h',
    'stdlib.h',
    'string.h',
]

if target_machine.system() != 'windows'
    _headers += [
        'arpa/inet.h',
        'fcntl.h',
        'netdb.h',
        'netinet/in.h',
        'sys/ioctl.h',
        'sys/socket.h',
        'sys/time.h',
        'dlfcn.h',
        'sys/stat.h',
        'sys/types.h',
    ]
elif cc.get_id() == 'msvc' or cc.get_id() == 'clang-cl'
    _headers += []
else
    _headers += [
        'strings.h',
        'unistd.h',
    ]
endif

foreach _header : _headers
    if cc.has_header(_header)
        c_defs += '-DHAVE_' + _header.underscorify().to_upper()
    endif
endforeach

_header_symbols = [
    [ 'sys/time.h', [ 'gettimeofday' ] ],
    [ 'time.h', [ 'gmtime_r', 'strftime' ] ],
    [ 'string.h', [ 'memset' ] ]
]

if target_machine.system() == 'windows'
    c_defs += [
        '-DHAVE_GETHOSTBYNAME',
        '-DHAVE_GETADDRINFO',
        '-DHAVE_SOCKET'
    ]
else
    _header_symbols += [
        [ 'arpa/inet.h', [ 'inet_ntoa' ] ],
        [ 'netdb.h', [ 'gethostbyname', 'getaddrinfo' ] ],
        [ 'unistd.h', [ 'getpid' ] ],
    ]
endif

foreach _pair : _header_symbols
    foreach _symbol : _pair[1]
        if cc.has_function(_symbol, prefix: [ '#include<@0@>'.format(_pair[0]) ])
            c_defs += '-DHAVE_' + _symbol.underscorify().to_upper()
        endif
    endforeach
endforeach

if cc.has_function('__atomic_fetch_add')
    c_defs += '-DHAVE_C___ATOMIC'
endif

# type sizes
foreach _header, _types : {
    'time.h': [ 'time_t' ],
    '': [ 'long long', 'long' ]
}
    foreach _type : _types
        _prefix = []
        if _header != ''
            _prefix += '#include<@0@>'.format(_header)
        endif

        _size = cc.sizeof(_type, prefix: _prefix)
        if _size != -1
            c_defs += '-DSIZEOF_@0@=@1@'.format( _type.underscorify().to_upper(), _size)
        endif
    endforeach
endforeach

_uintptr_size = cc.sizeof('uintptr_t')
if _uintptr_size != -1
    c_defs += '-DHAVE_UINTPTR_T=@0@'.format(_uintptr_size)
endif

_uint128_size = cc.sizeof('__uint128_t')
if _uint128_size != -1
    c_defs += [
        '-D__UINT128_T=@0@'.format(_uint128_size),
        '-DHAVE___UINT128_T=@0@'.format(_uint128_size)
    ]
endif

# thread local storage
if cc.get_id() == 'msvc' or cc.get_id() == 'clang-cl'
    c_defs += '-DHAVE_THREAD_LS'
else
    foreach _keyword : [
        '__thread',
        '__declspec(thread)'
    ]
        if cc.compiles('void main(){static @0@ int x;}'.format(_keyword))
            c_defs += '-DHAVE_THREAD_LS'
            break
        endif
    endforeach
endif

# 16-bit compiler support
if target_machine.cpu_family() == 'dspic' or target_machine.cpu_family() == 'pic24'
    c_defs += '-DWC_16BIT_CPU'
endif

# check if assembler is available
assembler = 'none'
asm_opt = get_option('asm')
if asm_opt.allowed()
    if add_languages('masm', native: false, required: false)
        assembler = 'masm'
    endif

    if assembler != 'none'
        asm_opt = asm_opt.enable_if(true)
    endif
endif

# debugging
if get_option('debugging')
    c_args += [
        '-DDEBUG_WOLFSSL',
        '-DDEBUG'
    ]
endif

# threading
async_crypt_opt = get_option('async_crypt')
async_threads_opt = get_option('async_threads')

if get_option('single_threaded')
    c_defs += '-DSINGLE_THREADED'
    if async_crypt_opt.enabled()
        warning('overriding "async_crypt" with "single_threaded"')
    endif
    if async_threads_opt.enabled()
        warning('overriding "async_threads" with "single_threaded"')
    endif
    async_crypt_opt = disabler()
    async_threads_opt = disabler()
endif

has_threads = false

if async_crypt_opt.allowed() and async_threads_opt.allowed()
    threads_dep = dependency('threads', required: false)
    if threads_dep.found()
        async_threads_opt = async_threads_opt.enable_auto_if(true)
        async_crypt_opt = async_crypt_opt.enable_auto_if(true)
        has_threads = true
    endif
endif

if has_threads
    deps += threads_dep
    c_args += [
        '-DHAVE_PTHREAD',
        '-D_GNU_SOURCE',
    ]
else
    c_defs += '-DWC_NO_ASYNC_THREADING'
endif

# inline functions
inline_opt = get_option('inline')
if not inline_opt
    c_defs += '-DNO_INLINE'
endif

# hardening and rng
rng_opt = get_option('rng')
harden_opt = get_option('harden')

if not rng_opt
    c_defs += '-DWC_NO_RNG'
endif

if harden_opt
    c_args += [
        '-DTFM_TIMING_RESISTANT',
        '-DECC_TIMING_RESISTANT'
    ]

    if rng_opt
        c_defs += '-DWC_RSA_BLINDING'
    endif
else
    c_defs += '-DWC_NO_HARDEN'

    if cc.get_argument_syntax() == 'msvc'
        c_args += '/GS-'
    endif
endif

# low resource
low_resource_opt = get_option('low_resource')

if low_resource_opt
    c_args += [
        '-DUSE_SLOW_SHA',
        '-DUSE_SLOW_SHA256'
    ]

    if cc.get_id() == 'clang-cl'
        c_args += [ '/clang:-fno-vectorize', '/clang:-fno-unroll-loops' ]
    endif
endif

# not implemented options
lean_psk = false
lean_tls = false

# filesystem
filesystem_opt = get_option('filesystem')

if not filesystem_opt.allowed()
    c_defs += '-DNO_FILESYSTEM'
else
    if lean_psk or lean_tls
        c_defs += '-DNO_FILESYSTEM'
        filesystem_opt = filesystem_opt.disable_if(true, error_message: 'filesystem can\'t be used with lean PSK or TLS')
    else
        filesystem_opt = filesystem_opt.enable_if(true)
    endif
endif

# error strings
error_strings_opt = get_option('error_strings')

if not error_strings_opt
    c_defs += '-DNO_ERROR_STRINGS'
else
    if lean_psk or lean_tls
        c_defs += '-DNO_ERROR_STRINGS'
        error_strings_opt = false
    endif
endif

# hash functions option
sha_opt = get_option('sha')
sha224_opt = get_option('sha224')
sha384_opt = get_option('sha384')
sha512_opt = get_option('sha512')
sha3_opt = get_option('sha3')
shake128_opt = get_option('shake128')
shake256_opt = get_option('shake256')
md4_opt = get_option('md4')
md5_opt = get_option('md5')

# tls extensions options
alpn_opt = get_option('alpn')
sni_opt = get_option('sni')
tlx_opt = get_option('tlx')
session_ticket_opt = get_option('session_ticket')
ticket_nonce_malloc_opt = get_option('ticket_nonce_malloc')
extended_master_opt = get_option('extended_master')
enc_then_mac_opt = get_option('enc_then_mac')
supported_curves = false
supported_curves_opt = get_option('supported_curves')

# public key cryptography options
rsa_opt = get_option('rsa')
oaep_opt = get_option('oaep')
rsa_pss_opt = get_option('rsa_pss')
dh_opt = get_option('dh')
dh_default_params_opt = get_option('dh_default_params')
dsa_opt = get_option('dsa')
ecc_opt = get_option('ecc')
eccshamir_opt = get_option('eccshamir')
ecccustcurves_opt = get_option('ecccustcurves')
curve25519_opt = get_option('curve25519')
ed25519_opt = get_option('ed25519')
curve448_opt = get_option('curve448')
ed448_opt = get_option('ed448')

# symmetric ciphers options
aes_opt = get_option('aes')
aes_small_tables_opt = get_option('aes_small_tables')
aes_no_unroll_opt = get_option('aes_no_unroll')
aes128_opt = get_option('aes128')
aes192_opt = get_option('aes192')
aes256_opt = get_option('aes256')
aescbc_opt = get_option('aescbc')
aesgcm_opt = get_option('aesgcm')
aesccm_opt = get_option('aesccm')
aesctr_opt = get_option('aesctr')
aesofb_opt = get_option('aesofb')
aescfb_opt = get_option('aescfb')
aessiv_opt = get_option('aessiv')
aeskeywrap_opt = get_option('aeskeywrap')
des3_opt = get_option('des3')
des3_tls_suites_opt = get_option('des3_tls_suites')
arc4_opt = get_option('arc4')
aria_opt = get_option('aria')
chacha_opt = get_option('chacha')
poly1305_opt = get_option('poly1305')
cmac_opt = get_option('cmac')
hkdf_opt = get_option('hkdf')

aes_direct = false
keying_material = false

# tls/ssl protocol features options
tls13_opt = get_option('tls13')
tlsv12_opt = get_option('tlsv12')
old_tls_opt = get_option('old_tls')
dtls_opt = get_option('dtls')
dtls13_opt = get_option('dtls13')
dtls_cid_opt = get_option('dtls_cid')
srtp_opt = get_option('srtp')
postauth_opt = get_option('postauth')
hrr_cookie_opt = get_option('hrr_cookie')

dtls = dtls_opt.enabled()
hrr_cookie = hrr_cookie_opt.enabled()

# callbacks and hooks options
ex_data_opt = get_option('ex_data')
cryptocb_opt = get_option('cryptocb')
cryptocb_no_sw_test_opt = get_option('cryptocb_no_sw_test')
pkcallbacks_opt = get_option('pkcallbacks')

# certificate and PKI features options
crl_opt = get_option('crl')
ocsp_opt = get_option('ocsp')
ocspstapling_opt = get_option('ocspstapling')
ocspstapling_v2_opt = get_option('ocspstapling_v2')
alt_cert_chains_opt = get_option('alt_cert_chains')
ip_alt_name_opt = get_option('ip_alt_name')
certgen_opt = get_option('certgen')
certreq_opt = get_option('certreq')
certext_opt = get_option('certext')
certgencache_opt = get_option('certgencache')
keygen_opt = get_option('keygen')
enckeys_opt = get_option('enckeys')
pkcs12_opt = get_option('pkcs12')

# compatibility options
openssl_extra_opt = get_option('openssl_extra')
openssl_all_opt = get_option('openssl_all')
openssl_no_stub_opt = get_option('openssl_no_stub')
old_names_opt = get_option('old_names')

# meta switches
if get_option('quic')
    alpn_opt = alpn_opt.enable_if(true, error_message: 'QUIC support can\'t be enabled without ALPN')
    openssl_extra_opt = openssl_extra_opt.enable_if(true, error_message: 'QUIC support can\'t be enabled without OpenSSl extra compatibility')
    aesctr_opt = aesctr_opt.enable_if(true, error_message: 'QUIC support can\'t be enabled without AES CTR')
    curve25519 = 'enabled'
    sni_opt = sni_opt.enable_if(true, error_message: 'QUIC support can\'t be enabled without SNI')
endif

if get_option('curl')
    md4_opt = md4_opt.enable_if(true, error_message: 'curl support can\'t be enabled without MD4')
    des3_opt = des3_opt.enable_if(true, error_message: 'curl support can\'t be enabled without 3DES')
    alpn_opt = alpn_opt.enable_if(true, error_message: 'curl support can\'t be enabled without ALPN')
    ex_data_opt = ex_data_opt.enable_if(true, error_message: 'curl support can\'t be enabled without extra data support')
    openssl_extra_opt = openssl_extra_opt.enable_if(true, error_message: 'curl support can\'t be enabled without OpenSSL extra compatibility')
    crl_opt = 'enabled'
    ocsp_opt = ocsp_opt.enable_if(true, error_message: 'curl support can\'t be enabled without OCSP')
    ocspstapling_opt = ocspstapling_opt.enable_if(true, error_message: 'curl support can\'t be enabled wiothout OCSP stapling')
    ocspstapling_v2_opt = ocspstapling_v2_opt.enable_if(true, error_message: 'curl support can\'t be enabled without OCSP stapling v2')
    sni_opt = sni_opt.enable_if(true, error_message: 'curl support can\'t be enabled without SNI')
    alt_cert_chains_opt = alt_cert_chains_opt.enable_if(true, error_message: 'curl support can\'t be enabled without IP alternative certificate chains')
    ip_alt_name_opt = ip_alt_name_opt.enable_if(true, error_message: 'curl support can\'t be enabled without IP alternative names')
    session_ticket_opt = session_ticket_opt.enable_if(true, error_message: 'curl support can\'t be enabled without session tickets')
endif

# compatibility
if openssl_extra_opt.enabled()
    c_defs += '-DOPENSSL_EXTRA'
endif

if openssl_all_opt.enabled()
    c_args += [
        '-DOPENSSL_ALL',
        '-DWOLFSSL_EITHER_SIDE',
        '-DWC_RSA_NO_PADDING',
        '-DWOLFSSL_PSS_LONG_SALT',
        '-DWOLFSSL_TICKET_HAVE_ID',
        '-DWOLFSSL_ERROR_CODE_OPENSSL',
        '-DWOLFSSL_CERT_NAME_ALL'
    ]
    rsa_pss_opt = rsa_pss_opt.enable_if(true, error_message: 'OpenSSl all compatibility requires RSA PSS')
endif

if openssl_no_stub_opt
    c_defs += '-DNO_WOLFSSL_STUB'
endif

if old_names_opt
    c_args += [
        '-DNO_OLD_RNGNAME',
        '-DNO_OLD_WC_NAMES',
        '-DNO_OLD_SSL_NAMES',
        '-DNO_OLD_SHA_NAMES',
    ]
endif

# server and client
server_opt = get_option('server')
client_opt = get_option('client')

if not server_opt
    c_defs += '-DNO_WOLFSSL_SERVER'
endif

if not client_opt
    c_defs += '-DNO_WOLFSSL_CLIENT'
endif

# system ca certificates
sys_ca_certs_opt = get_option('sys_ca_certs')

if sys_ca_certs_opt.enabled()
    if target_machine.kernel() == 'xnu'
        _has_sec_trust_settings = false
        if cc.has_header('Security/SecTrustSettings.h')
            c_defs += '-DHAVE_SECURITY_SECTRUSTSETTINGS_H'
            _has_sec_trust_settings = true
        endif

        _sec_headers_count = 0
        foreach _header : [
            'Security/SecCertificate.h',
            'Security/SecTrust.h',
            'Security/SecPolicy.h'
        ]
            if cc.has_header(_header)
                c_defs += '-DHAVE_' + _header.underscorify().to_upper()
            endif
            _sec_headers_count = _sec_headers_count + 1
        endforeach

        if _has_sec_trust_settings or _sec_headers_count == 3
            if not _has_sec_trust_settings and _sec_headers_count == 3
                c_defs += '-DWOLFSSL_APPLE_NATIVE_CERT_VALIDATION'
            endif
        else
            error('Can\'t enable system CA certs without Apple Security.framework headers.')
        endif
    elif target_machine.kernel() == 'nt'
        filesystem_opt.enable_auto_if(true)
        assert(filesystem_opt.enabled(), 'Can\'t enable system CA certs without a filesystem')
    endif

    c_defs += '-DWOLFSSL_SYS_CA_CERTS'
endif

# malloc
if get_option('no_malloc')
    c_defs += '-DWOLFSSL_NO_MALLOC'
endif

# certificate and PKI features
if crl_opt != 'disabled'
    if crl_opt == 'io'
        c_defs += '-DHAVE_CRL_IO'
    endif

    c_defs += '-DHAVE_CRL'
endif

if ocspstapling_opt.enabled()
    c_defs += '-DHAVE_CERTIFICATE_STATUS_REQUEST'
    tlx_opt = tlx_opt.enable_if(true, error_message: 'OCSP stapling requires TLS extensions')
    ocsp_opt = ocsp_opt.enable_if(true, error_message: 'OCSP stapling requires OCSP')
endif

if ocspstapling_v2_opt.enabled()
    c_defs += '-DHAVE_CERTIFICATE_STATUS_REQUEST_V2'
    tlx_opt = tlx_opt.enable_if(true, error_message: 'OCSP stapling v2 requires TLS extensions')
    ocsp_opt = ocsp_opt.enable_if(true, error_message: 'OCSP stapling v2 requires OCSP')
endif

if ocsp_opt.enabled()
    c_defs += '-DHAVE_OCSP'
endif

if alt_cert_chains_opt.enabled()
    c_defs += '-DWOLFSSL_ALT_CERT_CHAINS'
endif

if ip_alt_name_opt.enabled()
    c_defs += '-DWOLFSSL_IP_ALT_NAME'
endif

if certgen_opt.enabled()
    c_defs += '-DWOLFSSL_CERT_GEN'
endif

if certreq_opt.enabled()
    c_defs += '-DWOLFSSL_CERT_REQ'
endif

if certext_opt.enabled()
    c_defs += '-DWOLFSSL_CERT_EXT'
endif

if certgencache_opt
    c_defs += '-DWOLFSSL_CERT_GEN_CACHE'
endif

if keygen_opt
    c_defs += '-DWOLFSSL_KEY_GEN'
endif

if not enckeys_opt \
    and ( \
        openssl_extra_opt.enabled()
        # or webserver_opt
        # or wpas_opt
        )
    enckeys_opt = true
endif

if enckeys_opt
    c_defs += '-DWOLFSSL_ENCRYPTED_KEYS'
endif

if not pkcs12_opt
    c_defs += '-DNO_PKCS12'
endif

# callbacks and hooks
if ex_data_opt.enabled()
    c_defs += '-DHAVE_EX_DATA'
endif

if cryptocb_opt
    c_defs += '-DWOLF_CRYPTO_CB'
endif

if cryptocb_no_sw_test_opt
    c_defs += '-DWC_TEST_NO_CRYPTOCB_SW_TEST'
endif

if pkcallbacks_opt
    c_defs += '-DHAVE_PK_CALLBACKS'
endif

# tls/ssl protocol features
if tls13_opt.enabled()
    supported_curves = true
    tlx_opt = tlx_opt.enable_if(true, error_message: 'TLS 1.3 requires TLS extensions')
    c_defs += '-DWOLFSSL_TLS13'
endif

if not tlsv12_opt.enabled()
    c_defs += '-DWOLFSSL_NO_TLS12'
    old_tls_opt = old_tls_opt.disable_auto_if(true)
endif

if srtp_opt.enabled()
    c_defs += '-DWOLFSSL_SRTP'
    dtls = true
    keying_material = true
endif

if dtls
    c_defs += '-DWOLFSSL_DTLS'
endif

if dtls13_opt.enabled()
    assert(dtls, 'DTLS13 requires DTLS')
    assert(tls13_opt.enabled(), 'DTLS13 requires TLS13')

    c_args += [
        '-DWOLFSSL_DTLS13',
        '-DWOLFSSL_W64_WRAPPER'
    ]

    if hrr_cookie_opt.auto()
        warning('DTLS1.3 is enabled - enabling HRR Cookie')
        hrr_cookie = true
    endif

    if aes_opt.enabled()
        aes_direct = true
    endif
endif

if dtls_cid_opt.enabled()
    assert(dtls_opt.enabled(), 'CID are supported only for DTLSv1.3')
    c_defs += '-DWOLFSSL_DTLS_CID'
endif

if postauth_opt.enabled()
    if not tls13_opt.enabled()
        warning('TLS 1.3 is disabled - disabling Post-handshake Authentication')
    else
        c_defs += '-DWOLFSSL_POST_HANDSHAKE_AUTH'
    endif
endif

if hrr_cookie
    if not tls13_opt.enabled()
        warning('TLS 1.3 is disabled - disabling HRR Cookie')
        hrr_cookie = false
    else
        c_defs += '-DWOLFSSL_SEND_HRR_COOKIE'
    endif
endif

# symmetric ciphers
if not aes_opt.enabled()
    c_defs += '-DNO_AES'
    # assert(fortress_opt, 'fortress requires aes')
    # assert(ecc_encrypt_opt, 'cannot enable eccencrypt and hkdf without aes.')
else
    if lean_psk
        aes_opt = aes_opt.disable_if(true, 'AES is disabled due to lean PSK')
        c_defs += '-DNO_AES'
    endif
endif

if aes_opt.enabled()
    if not aes128_opt.enabled()
        c_defs += '-DNO_AES_128'
    endif

    if not aes192_opt.enabled()
        c_defs += '-DNO_AES_192'
    endif

    if not aes256_opt.enabled()
        c_defs += '-DNO_AES_256'
    endif

    aes_no_unroll_opt = aes_no_unroll_opt.enable_auto_if(low_resource_opt)
    if aes_no_unroll_opt.enabled()
        c_defs += '-DWOLFSSL_AES_NO_UNROLL'
    endif

    aes_small_tables_opt = aes_small_tables_opt.enable_auto_if(low_resource_opt)
    if aes_small_tables_opt.enabled()
        c_defs += '-DWOLFSSL_AES_SMALL_TABLES'
    endif
endif

if not aescbc_opt.enabled()
    c_defs += '-DNO_AES_CBC'
endif

if aesgcm_opt != 'disabled' and target_machine.endian() == 'little'
    aesgcm_opt = '4bit'
endif

if aesgcm_opt != 'disabled'
    assert(aesgcm_opt != 'disabled', 'AESGCM requires AES.')
    assert(rng_opt, 'AESGCM requires RNG')

    if aesgcm_opt == 'word32'
        c_defs += '-DGCM_WORD32'
    endif

    if aesgcm_opt == 'small' # or low_resource_opt
        c_defs += '-DGCM_SMALL'
    endif

    if aesgcm_opt == 'table'
        c_defs += '-DGCM_TABLE'
    endif

    if aesgcm_opt == '4bit'
        c_defs += '-DGCM_TABLE_4BIT'
    endif

    c_defs += '-DHAVE_AESGCM'
endif

if aesccm_opt.enabled()
    aesccm_opt.require(aes_opt.enabled(), error_message: 'AESCCM requires AES')
    c_defs += '-DHAVE_AESCCM'
endif

if aesctr_opt.enabled() # and not fortress_opt.enabled()
    aesctr_opt.require(aes_opt.enabled(), error_message: 'AESCTR requires AES')
    c_defs += '-DWOLFSSL_AES_COUNTER'
    aes_direct = true
endif

if aesofb_opt.enabled()
    c_defs += '-DWOLFSSL_AES_OFB'
    aes_direct = true
endif

if aescfb_opt.enabled()
    c_defs += '-DWOLFSSL_AES_CFB'
endif

if aessiv_opt.enabled()
    c_defs += '-DWOLFSSL_AES_SIV'
endif

if aeskeywrap_opt.enabled()
    c_defs += '-DHAVE_AES_KEYWRAP'
    aes_direct = true
endif

if not des3_opt.enabled()
    c_defs += '-DNO_DES3'
else
    if lean_psk or lean_tls
        warning('3DES was disabled due to lean PSK or TLS')
        c_defs += '-DNO_DES3'
    endif
endif

if not des3_tls_suites_opt.enabled()
    c_defs += '-DNO_DES3_TLS_SUITES'
endif

if not arc4_opt.enabled()
    c_defs += '-DNO_RC4'
else
    if lean_psk or lean_tls
        c_defs += '-DNO_RC4'
    endif
endif

if aria_opt.enabled()
    c_defs += '-DHAVE_ARIA'
endif

if chacha_opt != 'disabled' and (assembler != 'none' or chacha_opt == 'noasm')
    c_defs += '-DHAVE_CHACHA'

    if chacha_opt == 'noasm' or assembler == 'none'
        c_defs += '-DNO_CHACHA_ASM'
    endif
endif

if poly1305_opt.enabled()
    c_args += [
        '-DHAVE_POLY1305',
        '-DHAVE_ONE_TIME_AUTH'
    ]
endif

if cmac_opt.enabled()
    assert(aes, 'Cannot use CMAC without AES.')
    c_defs += '-DWOLFSSL_CMAC'
    aes_direct = true
endif

if tls13_opt.enabled()
    hkdf_opt = hkdf_opt.enable_if(true, error_message: 'TLS 1.3 can\'t be enabled without HKDF')
endif

if hkdf_opt.enabled()
    c_defs += '-DHAVE_HKDF'
endif

if aes_direct
    c_defs += '-DWOLFSSL_AES_DIRECT'
endif

# ???
curve448 = false
ed448 = false
fe448 = false
ge448 = false

ffdhe2048 = false
ffdhe3072 = false
ffdhe4096 = false
ffdhe6144 = false
ffdhe8192 = false

if dh_opt != 'disabled' and (tls13_opt.enabled() or supported_curves)
    ffdhe2048 = true
endif

# public key cryptography
if not rsa_opt.enabled()
    c_defs += '-DNO_RSA'
else
    if lean_psk or lean_tls
        c_defs += '-DNO_RSA'
    endif
endif

if not oaep_opt.enabled()
    c_defs += '-DWC_NO_RSA_OAEP'
endif

if rsa_opt.enabled() and tls13_opt.enabled()
    rsa_pss_opt = rsa_pss_opt.enable_if(true, error_message: 'TLS 1.3 requires RSA PSS')
endif

if rsa_pss_opt.enabled()
    c_args += [
        '-DWC_RSA_PSS',
        '-DWOLFSSL_PSS_LONG_SALT'
    ]
endif

if dh_opt != 'disabled' and lean_psk and lean_tls
    dh = 'disabled'
endif

if dh_opt == 'disabled'
    c_defs += '-DNO_DH'
elif dh_opt == 'const'
    c_defs += '-DWOLFSSL_DH_CONST'
endif

if dh_opt != 'disabled' # or qt_opt.enabled()
    c_defs += '-DHAVE_DH_DEFAULT_PARAMS'
endif

if not dsa_opt.enabled() # and not openssh_opt.enabled()
    c_defs += '-DNO_DSA'
endif

if ecc_opt != 'disabled'
    c_args += [
        '-DHAVE_ECC',
        '-DTFM_ECC256'
    ]

    if eccshamir_opt.enabled() # and not low_resource_opt
        c_defs += '-DECC_SHAMIR'
    endif

    if ecc_opt == 'nonblock'
        c_defs += '-DWC_ECC_NONBLOCK'
    endif
endif

if ecccustcurves_opt != 'disabled'
    if ecccustcurves_opt == 'all'
        c_args += [
            '-DHAVE_ECC_SECPR2',
            '-DHAVE_ECC_SECPR3',
            '-DHAVE_ECC_BRAINPOOL',
            '-DHAVE_ECC_KOBLITZ',
            '-DHAVE_ECC_CDH',
        ]
    endif

    c_defs += '-DWOLFSSL_CUSTOM_CURVES'
endif

if ed25519_opt != 'disabled'
    if ed25519_opt == 'small'
        assert(curve25519_opt != 'disabled')
        curve25519_opt = 'small'
    endif

    assert(sha512_opt.enabled(), 'cannot enable ed25519 without enabling sha512.')
    c_defs += '-DHAVE_ED25519'
endif

if curve25519_opt != 'disabled'
    if curve25519_opt == 'small' # or low_resource_opt
        c_defs += '-DCURVE25519_SMALL'
    elif curve25519_opt == 'no128bit'
        c_defs += '-DNO_CURVED25519_128BIT'
    endif

    c_defs += '-DHAVE_CURVE25519'
endif

if ed448_opt != 'disabled'
    if ed448_opt == 'small' # or low_resource_opt
        c_defs += '-DED448_SMALL'
        curve448_opt = 'small'
    endif

    assert(sha512_opt.enabled(), 'cannot enable ed448 without enabling sha512.')
    fe448 = true
    ge448 = true
    c_defs += '-DHAVE_ED448'

    # EdDSA448 requires SHAKE256 which requires SHA-3
    if shake256_opt == 'disabled'
        shake256_opt = ed448_opt == 'small' ? 'small' : 'enabled'
    endif
endif

if curve448_opt != 'disabled'
    if curve448_opt == 'small' # or low_resource_opt
        c_defs += '-DCURVE448_SMALL'
    endif

    if curve448_opt == 'no128bit' # or WOLFSSL_32BIT
        c_defs += '-DNO_CURVED448_128BIT'
    endif

    c_defs += '-DHAVE_CURVE448'
endif

# ???
if curve448
    src += files('wolfcrypt/src/curve448.c')
endif

if ed448
    src += files('wolfcrypt/src/ed448.c')
endif

if ge448
    src += files('wolfcrypt/src/ge_448.c')
    fe448 = true
endif

if fe448
    src += files('wolfcrypt/src/fe_448.c')
endif

if ffdhe2048
    c_defs += '-DHAVE_FFDHE_2048'
endif

if ffdhe3072
    c_defs += '-DHAVE_FFDHE_3072'
endif

if ffdhe4096
    c_defs += '-DHAVE_FFDHE_4096'
endif

if ffdhe6144
    c_defs += '-DHAVE_FFDHE_6144'
endif

if ffdhe8192
    c_defs += '-DHAVE_FFDHE_8192'
endif

# tls extensions
if tlx_opt.enabled()
    sni_opt = sni_opt.enable_if(true, error_message: 'TLS extensions require SNI')
    alpn_opt = alpn_opt.enable_if(true, error_message: 'TLS extensions require ALPN')
endif

if alpn_opt.enabled()
    c_defs += '-DHAVE_ALPN'
    tlx_opt = tlx_opt.enable_if(true, error_message: 'ALPN requires TLS extensions')
endif

if sni_opt.enabled()
    c_defs += '-DHAVE_SNI'
    tlx_opt = tlx_opt.enable_if(true, error_message: 'SNI requires TLS extensions')
endif

if session_ticket_opt.enabled()
    c_defs += '-DHAVE_SESSION_TICKET'
    tlx_opt = tlx_opt.enable_if(true, error_message: 'Session ticket requires TLS extensions')
endif

if ticket_nonce_malloc_opt.enabled()
    c_defs += '-DWOLFSSL_TICKET_NONCE_MALLOC'
endif

if extended_master_opt.enabled()
    c_defs += '-DHAVE_EXTENDED_MASTER'
endif

if enc_then_mac_opt.enabled()
    c_defs += '-DHAVE_ENCRYPT_THEN_MAC'
endif

if supported_curves_opt.enabled()
    if not (not ecc_opt.enabled() and not curve25519_opt.enabled() and not curve448_opt.enabled())
        supported_curves = true
    endif
endif

if supported_curves
    c_defs += '-DHAVE_SUPPORTED_CURVES'
    tlx_opt = tlx_opt.enable_if(true, error_message: 'Supported elliptic curves extensions require TLS extensions')
endif

if tlx_opt.enabled()
    tlx_opt = tlx_opt.enable_auto_if(true)
    c_args += [
        '-DHAVE_TLS_EXTENSIONS',
        '-DHAVE_MAX_FRAGMENT',
        '-DHAVE_TRUNCATED_HMAC',
        '-DHAVE_TRUSTED_CA'
    ]
endif

# hash functions
if not sha_opt.enabled() and not (lean_psk or lean_tls)
    c_args += [ '-DNO_SHA' ]
    sha_opt = false
    old_tls_opt = old_tls_opt.disable_auto_if(true)
endif

if sha224_opt.enabled()
    c_defs += '-DWOLFSSL_SHA224'
endif

if sha384_opt.enabled()
    c_defs += '-DWOLFSSL_SHA384'
endif

if sha512_opt.enabled()
    c_defs += '-DWOLFSSL_SHA512'
endif

if sha3_opt.enabled()
    c_defs += '-DWOLFSSL_SHA3'
endif

if not shake128_opt.enabled()
    c_defs += '-DWOLFSSL_NO_SHAKE128'
else
    if true # NOT WOLFSSL_32BIT
        assert(sha3_opt.enabled(), 'SHAKE128 requires SHA-3')
        c_defs += '-DWOLFSSL_SHAKE128'
    endif
endif

if not shake256_opt.enabled()
    c_defs += '-DWOLFSSL_NO_SHAKE256'
else
    if true # NOT WOLFSSL_32BIT
        assert(sha3_opt.enabled(), 'SHAKE256 requires SHA-3')
        c_defs += '-DWOLFSSL_SHAKE256'
    endif
endif

if not md4_opt.enabled()
    c_defs += '-DNO_MD4'
endif

if not md5_opt.enabled()
    if md5_opt.auto() \
        and ( \
            old_tls_opt.enabled() \
            or des3_opt.enabled()\
            or openssl_all.enabled() \
        )
        md5_opt = md5_opt.enable_if(true)
    else
        c_defs += '-DNO_MD5'
    endif
endif

# old tls
if not old_tls_opt.enabled()
    c_defs += '-DNO_OLD_TLS'
endif

adead_only = false
if (tls13_opt.enabled() and not tlsv12_opt.enabled()) \
    or (not aescbc_opt.enabled() and not des3_opt.enabled() and not arc4_opt.enabled()
        # and not camellia_opt.enabled() and not null_cipher.enabled()
        )
    adead_only = true
endif

if adead_only and old_tls_opt.enabled()
    error('must enable any non-AED cipher to enable old tls')
endif

c_args += c_defs

cdata = configuration_data()
foreach _def : c_defs
    _vals = _def.replace('-D', '').split('=')
    cdata.set(_vals[0], _vals.length() != 1 ? _vals[1] : 1)
endforeach

cfg_file = configure_file(
    output: 'user_settings.h',
    configuration: cdata,
)

message(cfg_file.full_path())

src += files(
    'src/crl.c',
    'src/dtls13.c',
    'src/dtls.c',
    'src/internal.c',
    'src/wolfio.c',
    'src/keys.c',
    'src/ocsp.c',
    'src/ssl.c',
    'src/tls.c',
    'wolfcrypt/src/aes.c',
    'wolfcrypt/src/arc4.c',
    'wolfcrypt/src/asn.c',
    'wolfcrypt/src/blake2b.c',
    'wolfcrypt/src/blake2s.c',
    'wolfcrypt/src/camellia.c',
    'wolfcrypt/src/chacha.c',
    'wolfcrypt/src/chacha20_poly1305.c',
    'wolfcrypt/src/cmac.c',
    'wolfcrypt/src/coding.c',
    'wolfcrypt/src/curve25519.c',
    'wolfcrypt/src/curve448.c',
    'wolfcrypt/src/cpuid.c',
    'wolfcrypt/src/cryptocb.c',
    'wolfcrypt/src/des3.c',
    'wolfcrypt/src/dilithium.c',
    'wolfcrypt/src/dh.c',
    'wolfcrypt/src/dsa.c',
    'wolfcrypt/src/ecc.c',
    'wolfcrypt/src/ed25519.c',
    'wolfcrypt/src/ed448.c',
    'wolfcrypt/src/error.c',
    'wolfcrypt/src/ext_mlkem.c',
    'wolfcrypt/src/falcon.c',
    'wolfcrypt/src/fe_448.c',
    'wolfcrypt/src/fe_low_mem.c',
    'wolfcrypt/src/fe_operations.c',
    'wolfcrypt/src/ge_448.c',
    'wolfcrypt/src/ge_low_mem.c',
    'wolfcrypt/src/ge_operations.c',
    'wolfcrypt/src/hash.c',
    'wolfcrypt/src/hmac.c',
    'wolfcrypt/src/hpke.c',
    'wolfcrypt/src/integer.c',
    'wolfcrypt/src/kdf.c',
    'wolfcrypt/src/wc_mlkem.c',
    'wolfcrypt/src/wc_mlkem_poly.c',
    'wolfcrypt/src/logging.c',
    'wolfcrypt/src/md2.c',
    'wolfcrypt/src/md4.c',
    'wolfcrypt/src/md5.c',
    'wolfcrypt/src/memory.c',
    'wolfcrypt/src/pkcs7.c',
    'wolfcrypt/src/pkcs12.c',
    'wolfcrypt/src/poly1305.c',
    'wolfcrypt/src/pwdbased.c',
    'wolfcrypt/src/random.c',
    'wolfcrypt/src/rc2.c',
    'wolfcrypt/src/ripemd.c',
    'wolfcrypt/src/rsa.c',
    'wolfcrypt/src/sha.c',
    'wolfcrypt/src/signature.c',
    'wolfcrypt/src/sphincs.c',
    'wolfcrypt/src/sp_c32.c',
    'wolfcrypt/src/sp_c64.c',
    'wolfcrypt/src/sp_int.c',
    'wolfcrypt/src/sp_x86_64.c',
    'wolfcrypt/src/srp.c',
    'wolfcrypt/src/tfm.c',
    'wolfcrypt/src/wc_encrypt.c',
    'wolfcrypt/src/wc_pkcs11.c',
    'wolfcrypt/src/wc_port.c',
    'wolfcrypt/src/wolfmath.c',
    'wolfcrypt/src/wolfevent.c',
    'wolfcrypt/src/port/liboqs/liboqs.c',
)

if tls13_opt.enabled()
    src += files('src/tls13.c')
endif

if not inline_opt
    src += files('wolfcrypt/src/misc.c')
endif

asm_sha = false
asm_aes = false
asm_chacha = false

if assembler == 'masm'
    if aes_opt.enabled() and target_machine.cpu() == 'amd64'
        src += files(
            'wolfcrypt/src/aes_asm.asm',
            'wolfcrypt/src/aes_xts_asm.asm',
        )

        if aesgcm_opt.enabled()
            src += files('wolfcrypt/src/aes_gcm_asm.asm')
        endif

        asm_aes = true
    endif

    if chacha_opt != 'disabled' and chacha_opt != 'noasm'
        src += files('wolfcrypt/src/chacha_asm.asm')
    endif
endif

if not asm_sha
    src += files(
        'wolfcrypt/src/sha256.c',
        'wolfcrypt/src/sha3.c',
        'wolfcrypt/src/sha512.c'
    )
endif

if not asm_aes
    src += files(
        'wolfcrypt/src/aes.c'
    )
endif

if not asm_chacha
    src += files(
        'wolfcrypt/src/aes.c'
    )
endif

# headers
inc = include_directories(
    'wolfssl/',
    '.',
)

if target_machine.kernel() == 'nt'
    deps += [
        cc.find_library('ws2_32'),
        cc.find_library('crypt32'),
    ]
elif target_machine.kernel() == 'xnu'
    if sys_ca_certs_opt.enabled()
        deps += dependency(
            'appleframeworks',
            modules: [
                'CoreFoundation',
                'Security',
            ],
        )
    endif
else
    if dh and dh_opt != 'const'
        deps += dependency('m')
    endif
endif

if get_option('default_library') == 'static'
    wolfssl_lib = library(
        'wolfssl',
        src,
        c_args: c_args,
        dependencies: deps
    )
elif get_option('default_library') == 'shared'
    c_args += '-DWOLFSSL_DLL'
    wolfssl_lib = shared_library(
        'wolfssl',
        src,
        c_args: c_args,
        dependencies: deps
    )
endif

wolfssl_dep = declare_dependency(
    link_with: wolfssl_lib,
    include_directories: inc,
    dependencies: deps
)

crypt_test_c_args = c_args + [ '-DWOLFSSL_NO_OPTIONS_H' ]

if get_option('crypt_tests')
    executable(
        'wolfcrypttest',
        files(
            'wolfcrypt/test/test.c'
        ),
        dependencies: [ wolfssl_dep ],
        c_args: crypt_test_c_args
    )

    executable(
        'wolfcryptbench',
        files(
            'wolfcrypt/benchmark/benchmark.c'
        ),
        dependencies: [ wolfssl_dep ],
        c_args: crypt_test_c_args
    )
endif

if get_option('testsuite')
    assert(client_opt, 'testsuite requires client')
    assert(server_opt, 'testsuite requires server')
    assert(tls13_opt.enabled() or tlsv12_opt.enabled() or old_tls_opt.enabled(), 'testsuite requires tls')

    executable(
        'testsuite',
        files(
            'examples/client/client.c',
            'examples/server/server.c',
            'examples/echoclient/echoclient.c',
            'examples/echoserver/echoserver.c',
            'wolfcrypt/test/test.c',
            'testsuite/testsuite.c',
        ),
        dependencies: [ wolfssl_dep ],
        c_args: crypt_test_c_args + [ '-DWOLFSSL_USER_SETTINGS', '-DNO_MAIN_DRIVER' ]
    )
endif

if get_option('examples')
    warning(client_opt or server_opt, 'examples specified without server or client options')

    if client_opt
        executable(
            'client',
            files('examples/client/client.c'),
            dependencies: [ wolfssl_dep ],
            c_args: crypt_test_c_args + [ '-DWOLFSSL_USER_SETTINGS' ]
        )

        executable(
            'echoclient',
            files('examples/echoclient/echoclient.c'),
            dependencies: [ wolfssl_dep ],
            c_args: crypt_test_c_args + [ '-DWOLFSSL_USER_SETTINGS' ]
        )
    endif

    if server_opt
        executable(
            'server',
            files('examples/server/server.c'),
            dependencies: [ wolfssl_dep ],
            c_args: crypt_test_c_args + [ '-DWOLFSSL_USER_SETTINGS' ]
        )

        executable(
            'echoserver',
            files('examples/echoserver/echoserver.c'),
            dependencies: [ wolfssl_dep ],
            c_args: crypt_test_c_args + [ '-DWOLFSSL_USER_SETTINGS' ]
        )
    endif
endif
