/****************************************************************************** * *  Copyright (C) 1999-2012 Broadcom Corporation * *  Licensed under the Apache License, Version 2.0 (the "License"); *  you may not use this file except in compliance with the License. *  You may obtain a copy of the License at: * *  http://www.apache.org/licenses/LICENSE-2.0 * *  Unless required by applicable law or agreed to in writing, software *  distributed under the License is distributed on an "AS IS" BASIS, *  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. *  See the License for the specific language governing permissions and *  limitations under the License. * ******************************************************************************/ #include "NfcAdaptation.h" #include <aidl/android/hardware/nfc/BnNfc.h> #include <aidl/android/hardware/nfc/BnNfcClientCallback.h> #include <aidl/android/hardware/nfc/INfc.h> #include <android-base/logging.h> #include <android-base/properties.h> #include <android-base/stringprintf.h> #include <android/binder_ibinder.h> #include <android/binder_manager.h> #include <android/binder_process.h> #include <android/hardware/nfc/1.1/INfc.h> #include <android/hardware/nfc/1.2/INfc.h> #include <cutils/properties.h> #include <hwbinder/ProcessState.h> #include <thread> #include "NfcVendorExtn.h" #include "debug_nfcsnoop.h" #include "nfa_api.h" #include "nfa_rw_api.h" #include "nfa_sys.h" #include "nfa_sys_int.h" #include "nfc_config.h" #include "nfc_int.h" using ::android::wp; using ::android::hardware::hidl_death_recipient; using ::android::hidl::base::V1_0::IBase; using android::OK; using android::sp; using android::status_t; using android::base::StringPrintf; using android::hardware::ProcessState; using android::hardware::Return; using android::hardware::Void; using android::hardware::nfc::V1_0::INfc; using android::hardware::nfc::V1_1::PresenceCheckAlgorithm; using INfcV1_1 = android::hardware::nfc::V1_1::INfc; using INfcV1_2 = android::hardware::nfc::V1_2::INfc; using NfcVendorConfigV1_1 = android::hardware::nfc::V1_1::NfcConfig; using NfcVendorConfigV1_2 = android::hardware::nfc::V1_2::NfcConfig; using android::hardware::nfc::V1_1::INfcClientCallback; using android::hardware::hidl_vec; using INfcAidl = ::aidl::android::hardware::nfc::INfc; using NfcAidlConfig = ::aidl::android::hardware::nfc::NfcConfig; using AidlPresenceCheckAlgorithm =    ::aidl::android::hardware::nfc::PresenceCheckAlgorithm; using INfcAidlClientCallback =    ::aidl::android::hardware::nfc::INfcClientCallback; using NfcAidlEvent = ::aidl::android::hardware::nfc::NfcEvent; using NfcAidlStatus = ::aidl::android::hardware::nfc::NfcStatus; using ::aidl::android::hardware::nfc::NfcCloseType; using Status = ::ndk::ScopedAStatus; #define VERBOSE_VENDOR_LOG_PROPERTY "persist.nfc.vendor_debug_enabled" #define DEFAULT_CRASH_LOGS_PATH "/data/misc/nfc/logs/hal_crash_logs" std::string NFC_AIDL_HAL_SERVICE_NAME = "android.hardware.nfc.INfc/default"; extern void GKI_shutdown(); extern void verify_stack_non_volatile_store(); extern void delete_stack_non_volatile_store(bool forceDelete); NfcAdaptation* NfcAdaptation::mpInstance = nullptr; ThreadMutex NfcAdaptation::sLock; ThreadCondVar NfcAdaptation::mHalOpenCompletedEvent; sp<INfc> NfcAdaptation::mHal; sp<INfcV1_1> NfcAdaptation::mHal_1_1; sp<INfcV1_2> NfcAdaptation::mHal_1_2; INfcClientCallback* NfcAdaptation::mCallback; std::shared_ptr<INfcAidlClientCallback> mAidlCallback; ::ndk::ScopedAIBinder_DeathRecipient mDeathRecipient; std::shared_ptr<INfcAidl> mAidlHal; int32_t mAidlHalVer; static NfcVendorExtn* sNfcVendorExtn = nullptr; static bool sVndExtnsPresent = false; bool nfc_nci_reset_keep_cfg_enabled = false; uint8_t nfc_nci_reset_type = 0x00; std::string nfc_storage_path; uint8_t appl_dta_mode_flag = 0x00; bool isDownloadFirmwareCompleted = false; bool use_aidl = false; uint8_t mute_tech_route_option = 0x00; unsigned int t5t_mute_legacy = 0; bool nfa_ee_route_debounce_timer = true; extern tNFA_DM_CFG nfa_dm_cfg; extern tNFA_PROPRIETARY_CFG nfa_proprietary_cfg; extern tNFA_HCI_CFG nfa_hci_cfg; extern uint8_t nfa_ee_max_ee_cfg; extern bool nfa_poll_bail_out_mode; // Whitelist for hosts allowed to create a pipe // See ADM_CREATE_PIPE command in the ETSI test specification // ETSI TS 102 622, section 6.1.3.1 static std::vector<uint8_t> host_allowlist; [[maybe_unused]] static int get_vsr_api_level() {  int vendor_api_level =      ::android::base::GetIntProperty("ro.vendor.api_level", -1);  if (vendor_api_level != -1) {    return vendor_api_level;  }   // Android S and older devices do not define ro.vendor.api_level  vendor_api_level = ::android::base::GetIntProperty("ro.board.api_level", -1);  if (vendor_api_level == -1) {    vendor_api_level =        ::android::base::GetIntProperty("ro.board.first_api_level", -1);  }   int product_api_level =      ::android::base::GetIntProperty("ro.product.first_api_level", -1);  if (product_api_level == -1) {    product_api_level =        ::android::base::GetIntProperty("ro.build.version.sdk", -1);  }   // VSR API level is the minimum of vendor_api_level and product_api_level.  if (vendor_api_level == -1 || vendor_api_level > product_api_level) {    return product_api_level;  }  return vendor_api_level; } static void notifyHalBinderDied() {  if (sVndExtnsPresent) {    uint8_t event = -1, status = -1;    sNfcVendorExtn->processEvent(event, status);  } } namespace { void initializeGlobalDebugEnabledFlag() {  bool nfc_debug_enabled =      (NfcConfig::getUnsigned(NAME_NFC_DEBUG_ENABLED, 0) != 0) ||      property_get_bool("persist.nfc.debug_enabled", true);   android::base::SetMinimumLogSeverity(nfc_debug_enabled ? android::base::DEBUG                                                         : android::base::INFO);   LOG(VERBOSE) << StringPrintf("%s: level=%u", __func__, nfc_debug_enabled); } // initialize NciResetType Flag // NCI_RESET_TYPE // 0x00 default, reset configurations every time. // 0x01, reset configurations only once every boot. // 0x02, keep configurations. void initializeNciResetTypeFlag() {  nfc_nci_reset_type = NfcConfig::getUnsigned(NAME_NCI_RESET_TYPE, 0);  LOG(VERBOSE) << StringPrintf("%s: nfc_nci_reset_type=%u", __func__,                             nfc_nci_reset_type); } // initialize MuteTechRouteOption Flag // MUTE_TECH_ROUTE_OPTION // 0x00: Default. Route mute techs to DH, enable block bit and set power state // to 0x00 0x01: Remove mute techs from rf discover cmd void initializeNfcMuteTechRouteOptionFlag() {  mute_tech_route_option =      NfcConfig::getUnsigned(NAME_MUTE_TECH_ROUTE_OPTION, 0);  LOG(VERBOSE) << StringPrintf("%s: mute_tech_route_option=%u", __func__,                               mute_tech_route_option); } // Abort nfc service when AIDL process died. void HalAidlBinderDied(void* /* cookie */) {  LOG(ERROR) << StringPrintf("%s: INfc aidl hal died, exiting procces to restart", __func__);  storeNfcSnoopLogs(DEFAULT_CRASH_LOGS_PATH, DEFAULT_NFCSNOOP_FILE_SIZE);  notifyHalBinderDied();  exit(0); } }  // namespace class NfcClientCallback : public INfcClientCallback { public:  NfcClientCallback(tHAL_NFC_CBACK* eventCallback,                    tHAL_NFC_DATA_CBACK dataCallback) {    mEventCallback = eventCallback;    mDataCallback = dataCallback;  };  virtual ~NfcClientCallback() = default;  Return<void> sendEvent_1_1(      ::android::hardware::nfc::V1_1::NfcEvent event,      ::android::hardware::nfc::V1_0::NfcStatus event_status) override {    if (sVndExtnsPresent) {      sNfcVendorExtn->processEvent((uint8_t)event,                                   (tHAL_NFC_STATUS)event_status);    }    mEventCallback((uint8_t)event, (tHAL_NFC_STATUS)event_status);    return Void();  };  Return<void> sendEvent(      ::android::hardware::nfc::V1_0::NfcEvent event,      ::android::hardware::nfc::V1_0::NfcStatus event_status) override {    if (sVndExtnsPresent) {      sNfcVendorExtn->processEvent((uint8_t)event,                                   (tHAL_NFC_STATUS)event_status);    }    mEventCallback((uint8_t)event, (tHAL_NFC_STATUS)event_status);    return Void();  };  Return<void> sendData(      const ::android::hardware::nfc::V1_0::NfcData& data) override {    ::android::hardware::nfc::V1_0::NfcData copy = data;    if (sVndExtnsPresent) {      bool isVndExtSpecRsp =          sNfcVendorExtn->processRspNtf(copy.size(), &copy[0]);      // If true to be consumed by vendor extension, otherwise need to be      // handled in libnfc-nci      if (isVndExtSpecRsp) {        return Void();      }    }    mDataCallback(copy.size(), &copy[0]);    return Void();  };  private:  tHAL_NFC_CBACK* mEventCallback;  tHAL_NFC_DATA_CBACK* mDataCallback; }; class NfcHalDeathRecipient : public hidl_death_recipient { public:  android::sp<android::hardware::nfc::V1_0::INfc> mNfcDeathHal;  NfcHalDeathRecipient(android::sp<android::hardware::nfc::V1_0::INfc>& mHal) {    mNfcDeathHal = mHal;  }   virtual void serviceDied(      uint64_t /* cookie */,      const wp<::android::hidl::base::V1_0::IBase>& /* who */) {    ALOGE(        "NfcHalDeathRecipient::serviceDied - Nfc-Hal service died. Killing "        "NfcService");    if (mNfcDeathHal) {      mNfcDeathHal->unlinkToDeath(this);    }    mNfcDeathHal = NULL;    notifyHalBinderDied();    exit(0);  }  void finalize() {    if (mNfcDeathHal) {      mNfcDeathHal->unlinkToDeath(this);    } else {      LOG(VERBOSE) << StringPrintf("%s: mNfcDeathHal is not set", __func__);    }     ALOGI("NfcHalDeathRecipient::destructor - NfcService");    mNfcDeathHal = NULL;  } }; class NfcAidlClientCallback    : public ::aidl::android::hardware::nfc::BnNfcClientCallback { public:  NfcAidlClientCallback(tHAL_NFC_CBACK* eventCallback,                        tHAL_NFC_DATA_CBACK dataCallback) {    mEventCallback = eventCallback;    mDataCallback = dataCallback;  };  virtual ~NfcAidlClientCallback() = default;   ::ndk::ScopedAStatus sendEvent(NfcAidlEvent event,                                 NfcAidlStatus event_status) override {    if (sVndExtnsPresent) {      bool isVndExtSpecEvt =          sNfcVendorExtn->processEvent((uint8_t)event, (uint8_t)event_status);      if (isVndExtSpecEvt) {        // If true to be handled only in extension,        // otherwise processed in libNfc        return ::ndk::ScopedAStatus::ok();      }    }    uint8_t e_num;    uint8_t s_num;    switch (event) {      case NfcAidlEvent::OPEN_CPLT:        e_num = HAL_NFC_OPEN_CPLT_EVT;        break;      case NfcAidlEvent::CLOSE_CPLT:        e_num = HAL_NFC_CLOSE_CPLT_EVT;        break;      case NfcAidlEvent::POST_INIT_CPLT:        e_num = HAL_NFC_POST_INIT_CPLT_EVT;        break;      case NfcAidlEvent::PRE_DISCOVER_CPLT:        e_num = HAL_NFC_PRE_DISCOVER_CPLT_EVT;        break;      case NfcAidlEvent::HCI_NETWORK_RESET:        e_num = HAL_HCI_NETWORK_RESET;        break;      case NfcAidlEvent::REQUEST_CONTROL:        e_num = HAL_NFC_REQUEST_CONTROL_EVT;        break;      case NfcAidlEvent::RELEASE_CONTROL:        e_num = HAL_NFC_RELEASE_CONTROL_EVT;        break;      case NfcAidlEvent::ERROR:      default:        e_num = HAL_NFC_ERROR_EVT;    }    switch (event_status) {      case NfcAidlStatus::OK:        s_num = HAL_NFC_STATUS_OK;        break;      case NfcAidlStatus::FAILED:        s_num = HAL_NFC_STATUS_FAILED;        break;      case NfcAidlStatus::ERR_TRANSPORT:        s_num = HAL_NFC_STATUS_ERR_TRANSPORT;        break;      case NfcAidlStatus::ERR_CMD_TIMEOUT:        s_num = HAL_NFC_STATUS_ERR_CMD_TIMEOUT;        break;      case NfcAidlStatus::REFUSED:        s_num = HAL_NFC_STATUS_REFUSED;        break;      default:        s_num = HAL_NFC_STATUS_FAILED;    }    mEventCallback(e_num, (tHAL_NFC_STATUS)s_num);    return ::ndk::ScopedAStatus::ok();  };  ::ndk::ScopedAStatus sendData(const std::vector<uint8_t>& data) override {    std::vector<uint8_t> copy = data;    if (sVndExtnsPresent) {      bool isVndExtSpecRsp =          sNfcVendorExtn->processRspNtf(copy.size(), &copy[0]);      // If true to be consumed by vendor extension, otherwise need to be      // handled in libnfc-nci      if (isVndExtSpecRsp) {        return ::ndk::ScopedAStatus::ok();      }    }    mDataCallback(copy.size(), &copy[0]);    return ::ndk::ScopedAStatus::ok();  };  private:  tHAL_NFC_CBACK* mEventCallback;  tHAL_NFC_DATA_CBACK* mDataCallback; }; /******************************************************************************* ** ** Function:    NfcAdaptation::NfcAdaptation() ** ** Description: class constructor ** ** Returns:     none ** *******************************************************************************/ NfcAdaptation::NfcAdaptation() {  memset(&mHalEntryFuncs, 0, sizeof(mHalEntryFuncs));  mDeathRecipient = ::ndk::ScopedAIBinder_DeathRecipient(      AIBinder_DeathRecipient_new(HalAidlBinderDied));  sNfcVendorExtn = NfcVendorExtn::getInstance(); } /******************************************************************************* ** ** Function:    NfcAdaptation::~NfcAdaptation() ** ** Description: class destructor ** ** Returns:     none ** *******************************************************************************/ NfcAdaptation::~NfcAdaptation() { mpInstance = nullptr; } /******************************************************************************* ** ** Function:    NfcAdaptation::GetInstance() ** ** Description: access class singleton ** ** Returns:     pointer to the singleton object ** *******************************************************************************/ NfcAdaptation& NfcAdaptation::GetInstance() {  AutoThreadMutex a(sLock);   if (!mpInstance) {    mpInstance = new NfcAdaptation;    mpInstance->InitializeHalDeviceContext();  }  return *mpInstance; } void NfcAdaptation::GetVendorConfigs(    std::map<std::string, ConfigValue>& configMap) {  NfcVendorConfigV1_2 configValue;  NfcAidlConfig aidlConfigValue;  if (mAidlHal) {    mAidlHal->getConfig(&aidlConfigValue);  } else if (mHal_1_2) {    mHal_1_2->getConfig_1_2(        [&configValue](NfcVendorConfigV1_2 config) { configValue = config; });  } else if (mHal_1_1) {    mHal_1_1->getConfig([&configValue](NfcVendorConfigV1_1 config) {      configValue.v1_1 = config;      configValue.defaultIsoDepRoute = 0x00;    });  }   if (mAidlHal) {    std::vector<int8_t> nfaPropCfg = {        aidlConfigValue.nfaProprietaryCfg.protocol18092Active,        aidlConfigValue.nfaProprietaryCfg.protocolBPrime,        aidlConfigValue.nfaProprietaryCfg.protocolDual,        aidlConfigValue.nfaProprietaryCfg.protocol15693,        aidlConfigValue.nfaProprietaryCfg.protocolKovio,        aidlConfigValue.nfaProprietaryCfg.protocolMifare,        aidlConfigValue.nfaProprietaryCfg.discoveryPollKovio,        aidlConfigValue.nfaProprietaryCfg.discoveryPollBPrime,        aidlConfigValue.nfaProprietaryCfg.discoveryListenBPrime};    if (mAidlHalVer > 1) {      nfaPropCfg.push_back(aidlConfigValue.nfaProprietaryCfg.protocolChineseId);    }    configMap.emplace(NAME_NFA_PROPRIETARY_CFG, ConfigValue(nfaPropCfg));    configMap.emplace(NAME_NFA_POLL_BAIL_OUT_MODE,                      ConfigValue(aidlConfigValue.nfaPollBailOutMode ? 1 : 0));    if (aidlConfigValue.offHostRouteUicc.size() != 0) {      configMap.emplace(NAME_OFFHOST_ROUTE_UICC,                        ConfigValue(aidlConfigValue.offHostRouteUicc));    }    if (aidlConfigValue.offHostRouteEse.size() != 0) {      configMap.emplace(NAME_OFFHOST_ROUTE_ESE,                        ConfigValue(aidlConfigValue.offHostRouteEse));    }    // AIDL byte would be int8_t in C++.    // Here we force cast int8_t to uint8_t for ConfigValue    configMap.emplace(        NAME_DEFAULT_OFFHOST_ROUTE,        ConfigValue((uint8_t)aidlConfigValue.defaultOffHostRoute));    configMap.emplace(NAME_DEFAULT_ROUTE,                      ConfigValue((uint8_t)aidlConfigValue.defaultRoute));    configMap.emplace(        NAME_DEFAULT_NFCF_ROUTE,        ConfigValue((uint8_t)aidlConfigValue.defaultOffHostRouteFelica));    configMap.emplace(NAME_DEFAULT_ISODEP_ROUTE,                      ConfigValue((uint8_t)aidlConfigValue.defaultIsoDepRoute));    configMap.emplace(        NAME_DEFAULT_SYS_CODE_ROUTE,        ConfigValue((uint8_t)aidlConfigValue.defaultSystemCodeRoute));    configMap.emplace(        NAME_DEFAULT_SYS_CODE_PWR_STATE,        ConfigValue((uint8_t)aidlConfigValue.defaultSystemCodePowerState));    configMap.emplace(NAME_OFF_HOST_SIM_PIPE_ID,                      ConfigValue((uint8_t)aidlConfigValue.offHostSIMPipeId));    configMap.emplace(NAME_OFF_HOST_ESE_PIPE_ID,                      ConfigValue((uint8_t)aidlConfigValue.offHostESEPipeId));    configMap.emplace(NAME_T4T_NFCEE_ENABLE,                      ConfigValue(aidlConfigValue.t4tNfceeEnable ? 1 : 0));     if (aidlConfigValue.offHostSimPipeIds.size() != 0) {      configMap.emplace(NAME_OFF_HOST_SIM_PIPE_IDS,                        ConfigValue(aidlConfigValue.offHostSimPipeIds));    }    configMap.emplace(NAME_ISO_DEP_MAX_TRANSCEIVE,                      ConfigValue(aidlConfigValue.maxIsoDepTransceiveLength));    if (aidlConfigValue.hostAllowlist.size() != 0) {      configMap.emplace(NAME_DEVICE_HOST_ALLOW_LIST,                        ConfigValue(aidlConfigValue.hostAllowlist));    }    /* For Backwards compatibility */    if (aidlConfigValue.presenceCheckAlgorithm ==        AidlPresenceCheckAlgorithm::ISO_DEP_NAK) {      configMap.emplace(NAME_PRESENCE_CHECK_ALGORITHM,                        ConfigValue((uint32_t)NFA_RW_PRES_CHK_ISO_DEP_NAK));    } else {      configMap.emplace(          NAME_PRESENCE_CHECK_ALGORITHM,          ConfigValue((uint32_t)aidlConfigValue.presenceCheckAlgorithm));    }  } else if (mHal_1_1 || mHal_1_2) {    std::vector<uint8_t> nfaPropCfg = {        configValue.v1_1.nfaProprietaryCfg.protocol18092Active,        configValue.v1_1.nfaProprietaryCfg.protocolBPrime,        configValue.v1_1.nfaProprietaryCfg.protocolDual,        configValue.v1_1.nfaProprietaryCfg.protocol15693,        configValue.v1_1.nfaProprietaryCfg.protocolKovio,        configValue.v1_1.nfaProprietaryCfg.protocolMifare,        configValue.v1_1.nfaProprietaryCfg.discoveryPollKovio,        configValue.v1_1.nfaProprietaryCfg.discoveryPollBPrime,        configValue.v1_1.nfaProprietaryCfg.discoveryListenBPrime};    configMap.emplace(NAME_NFA_PROPRIETARY_CFG, ConfigValue(nfaPropCfg));    configMap.emplace(NAME_NFA_POLL_BAIL_OUT_MODE,                      ConfigValue(configValue.v1_1.nfaPollBailOutMode ? 1 : 0));    configMap.emplace(NAME_DEFAULT_OFFHOST_ROUTE,                      ConfigValue(configValue.v1_1.defaultOffHostRoute));    if (configValue.offHostRouteUicc.size() != 0) {      configMap.emplace(NAME_OFFHOST_ROUTE_UICC,                        ConfigValue(configValue.offHostRouteUicc));    }    if (configValue.offHostRouteEse.size() != 0) {      configMap.emplace(NAME_OFFHOST_ROUTE_ESE,                        ConfigValue(configValue.offHostRouteEse));    }    configMap.emplace(NAME_DEFAULT_ROUTE,                      ConfigValue(configValue.v1_1.defaultRoute));    configMap.emplace(NAME_DEFAULT_NFCF_ROUTE,                      ConfigValue(configValue.v1_1.defaultOffHostRouteFelica));    configMap.emplace(NAME_DEFAULT_ISODEP_ROUTE,                      ConfigValue(configValue.defaultIsoDepRoute));    configMap.emplace(NAME_DEFAULT_SYS_CODE_ROUTE,                      ConfigValue(configValue.v1_1.defaultSystemCodeRoute));    configMap.emplace(        NAME_DEFAULT_SYS_CODE_PWR_STATE,        ConfigValue(configValue.v1_1.defaultSystemCodePowerState));    configMap.emplace(NAME_OFF_HOST_SIM_PIPE_ID,                      ConfigValue(configValue.v1_1.offHostSIMPipeId));    configMap.emplace(NAME_OFF_HOST_ESE_PIPE_ID,                      ConfigValue(configValue.v1_1.offHostESEPipeId));    configMap.emplace(NAME_ISO_DEP_MAX_TRANSCEIVE,                      ConfigValue(configValue.v1_1.maxIsoDepTransceiveLength));    if (configValue.v1_1.hostWhitelist.size() != 0) {      configMap.emplace(NAME_DEVICE_HOST_ALLOW_LIST,                        ConfigValue(configValue.v1_1.hostWhitelist));    }    /* For Backwards compatibility */    if (configValue.v1_1.presenceCheckAlgorithm ==        PresenceCheckAlgorithm::ISO_DEP_NAK) {      configMap.emplace(NAME_PRESENCE_CHECK_ALGORITHM,                        ConfigValue((uint32_t)NFA_RW_PRES_CHK_ISO_DEP_NAK));    } else {      configMap.emplace(          NAME_PRESENCE_CHECK_ALGORITHM,          ConfigValue((uint32_t)configValue.v1_1.presenceCheckAlgorithm));    }  }  if (sVndExtnsPresent) {    sNfcVendorExtn->getVendorConfigs(&configMap);  } } /******************************************************************************* ** ** Function:    NfcAdaptation::Initialize() ** ** Description: class initializer ** ** Returns:     none ** *******************************************************************************/ void NfcAdaptation::Initialize() {  const char* func = "NfcAdaptation::Initialize";  if (sVndExtnsPresent) {    sNfcVendorExtn->processEvent(HANDLE_NFC_ADAPTATION_INIT, HAL_NFC_STATUS_OK);  }  // Init log tag  android::base::InitLogging(nullptr);  android::base::SetDefaultTag("libnfc_nci");   initializeGlobalDebugEnabledFlag();  initializeNciResetTypeFlag();  initializeNfcMuteTechRouteOptionFlag();   LOG(VERBOSE) << StringPrintf("%s: enter", func);   nfc_storage_path = NfcConfig::getString(NAME_NFA_STORAGE, "/data/nfc");   if (NfcConfig::hasKey(NAME_NFA_DM_CFG)) {    std::vector<uint8_t> dm_config = NfcConfig::getBytes(NAME_NFA_DM_CFG);    if (dm_config.size() > 0) nfa_dm_cfg.auto_detect_ndef = dm_config[0];    if (dm_config.size() > 1) nfa_dm_cfg.auto_read_ndef = dm_config[1];    if (dm_config.size() > 2) nfa_dm_cfg.auto_presence_check = dm_config[2];    if (dm_config.size() > 3) nfa_dm_cfg.presence_check_option = dm_config[3];    // NOTE: The timeout value is not configurable here because the endianness    // of a byte array is ambiguous and needlessly difficult to configure.    // If this value needs to be configurable, a numeric config option should    // be used.  }   if (NfcConfig::hasKey(NAME_NFA_MAX_EE_SUPPORTED)) {    nfa_ee_max_ee_cfg = NfcConfig::getUnsigned(NAME_NFA_MAX_EE_SUPPORTED);    LOG(VERBOSE) << StringPrintf(        "%s: Overriding NFA_EE_MAX_EE_SUPPORTED to use %d", func,        nfa_ee_max_ee_cfg);  }   if (NfcConfig::hasKey(NAME_NFA_POLL_BAIL_OUT_MODE)) {    nfa_poll_bail_out_mode =        NfcConfig::getUnsigned(NAME_NFA_POLL_BAIL_OUT_MODE);    LOG(VERBOSE) << StringPrintf(        "%s: Overriding NFA_POLL_BAIL_OUT_MODE to use %d", func,        nfa_poll_bail_out_mode);  }   if (NfcConfig::hasKey(NAME_NFA_PROPRIETARY_CFG)) {    std::vector<uint8_t> p_config =        NfcConfig::getBytes(NAME_NFA_PROPRIETARY_CFG);    if (p_config.size() > 0)      nfa_proprietary_cfg.pro_protocol_18092_active = p_config[0];    if (p_config.size() > 1)      nfa_proprietary_cfg.pro_protocol_b_prime = p_config[1];    if (p_config.size() > 2)      nfa_proprietary_cfg.pro_protocol_dual = p_config[2];    if (p_config.size() > 3)      nfa_proprietary_cfg.pro_protocol_15693 = p_config[3];    if (p_config.size() > 4)      nfa_proprietary_cfg.pro_protocol_kovio = p_config[4];    if (p_config.size() > 5) nfa_proprietary_cfg.pro_protocol_mfc = p_config[5];    if (p_config.size() > 6)      nfa_proprietary_cfg.pro_discovery_kovio_poll = p_config[6];    if (p_config.size() > 7)      nfa_proprietary_cfg.pro_discovery_b_prime_poll = p_config[7];    if (p_config.size() > 8)      nfa_proprietary_cfg.pro_discovery_b_prime_listen = p_config[8];    if (p_config.size() > 9)      nfa_proprietary_cfg.pro_protocol_chinese_id = p_config[9];  }   // Configure allowlist of HCI host ID's  // See specification: ETSI TS 102 622, section 6.1.3.1  if (NfcConfig::hasKey(NAME_DEVICE_HOST_ALLOW_LIST)) {    host_allowlist = NfcConfig::getBytes(NAME_DEVICE_HOST_ALLOW_LIST);    nfa_hci_cfg.num_allowlist_host = host_allowlist.size();    nfa_hci_cfg.p_allowlist = &host_allowlist[0];  }   if (NfcConfig::hasKey(NAME_ISO15693_SKIP_GET_SYS_INFO_CMD)) {    t5t_mute_legacy =        NfcConfig::getUnsigned(NAME_ISO15693_SKIP_GET_SYS_INFO_CMD);  }   if (NfcConfig::hasKey(NAME_NFA_DM_LISTEN_ACTIVE_DEACT_NTF_TIMEOUT)) {    unsigned int value =        NfcConfig::getUnsigned(NAME_NFA_DM_LISTEN_ACTIVE_DEACT_NTF_TIMEOUT);    if (value > 0) {      nfa_dm_cfg.deact_ntf_listen_active_timeout = value * 1000;    }  }   if (NfcConfig::hasKey(NAME_NFA_EE_ROUTE_DEBOUNCE_TIMER)) {    if (NfcConfig::getUnsigned(NAME_NFA_EE_ROUTE_DEBOUNCE_TIMER) == 0x00) {      nfa_ee_route_debounce_timer = false;    }  }   verify_stack_non_volatile_store();  if (NfcConfig::hasKey(NAME_PRESERVE_STORAGE) &&      NfcConfig::getUnsigned(NAME_PRESERVE_STORAGE) == 1) {    LOG(VERBOSE) << StringPrintf("%s: preserve stack NV store", __func__);  } else {    delete_stack_non_volatile_store(FALSE);  }   GKI_init();  GKI_enable();  GKI_create_task((TASKPTR)NFCA_TASK, BTU_TASK, (int8_t*)"NFCA_TASK", nullptr, 0,                  (pthread_cond_t*)nullptr, nullptr);  {    AutoThreadMutex guard(mCondVar);    GKI_create_task((TASKPTR)Thread, MMI_TASK, (int8_t*)"NFCA_THREAD", nullptr, 0,                    (pthread_cond_t*)nullptr, nullptr);    mCondVar.wait();  }   debug_nfcsnoop_init();  LOG(VERBOSE) << StringPrintf("%s: exit", func); } /******************************************************************************* ** ** Function:    NfcAdaptation::Finalize() ** ** Description: class finalizer ** ** Returns:     none ** *******************************************************************************/ void NfcAdaptation::Finalize() {  const char* func = "NfcAdaptation::Finalize";  AutoThreadMutex a(sLock);   LOG(VERBOSE) << StringPrintf("%s: enter", func);  GKI_shutdown();   NfcConfig::clear();   if (mAidlHal != nullptr && AIBinder_isAlive(mAidlHal->asBinder().get())) {    if (sVndExtnsPresent) {      sNfcVendorExtn->finalize();    }    AIBinder_unlinkToDeath(mAidlHal->asBinder().get(), mDeathRecipient.get(),                           nullptr);    mAidlHal = nullptr;  } else if (mHal != nullptr) {    if (sVndExtnsPresent) {      sNfcVendorExtn->finalize();    }    mNfcHalDeathRecipient->finalize();  }  LOG(VERBOSE) << StringPrintf("%s: exit", func);  delete this; } void NfcAdaptation::FactoryReset() {  if (mAidlHal != nullptr) {    mAidlHal->factoryReset();  } else if (mHal_1_2 != nullptr) {    mHal_1_2->factoryReset();  } else if (mHal_1_1 != nullptr) {    mHal_1_1->factoryReset();  } } void NfcAdaptation::DeviceShutdown() {  if (mAidlHal != nullptr && AIBinder_isAlive(mAidlHal->asBinder().get())) {    mAidlHal->close(NfcCloseType::HOST_SWITCHED_OFF);    AIBinder_unlinkToDeath(mAidlHal->asBinder().get(), mDeathRecipient.get(),                           nullptr);    mAidlHal = nullptr;  } else {    if (mHal_1_2 != nullptr) {      mHal_1_2->closeForPowerOffCase();    } else if (mHal_1_1 != nullptr) {      mHal_1_1->closeForPowerOffCase();    }    if (mHal != nullptr) {      mHal->unlinkToDeath(mNfcHalDeathRecipient);    }  } } /******************************************************************************* ** ** Function:    NfcAdaptation::Dump ** ** Description: Native support for dumpsys function. ** ** Returns:     None. ** *******************************************************************************/ void NfcAdaptation::Dump(int fd) {  LOG(DEBUG) << StringPrintf("%s :enable_cplt_flags=0x%x, enable_cplt_mask=0x%x",                               __func__,                               nfa_sys_cb.enable_cplt_flags,                               nfa_sys_cb.enable_cplt_mask);  dprintf(fd, "enable_cplt_flags=0x%x, enable_cplt_mask=0x%x\n",          nfa_sys_cb.enable_cplt_flags,          nfa_sys_cb.enable_cplt_mask);  debug_nfcsnoop_dump(fd); } /******************************************************************************* ** ** Function:    NfcAdaptation::signal() ** ** Description: signal the CondVar to release the thread that is waiting ** ** Returns:     none ** *******************************************************************************/ void NfcAdaptation::signal() { mCondVar.signal(); } /******************************************************************************* ** ** Function:    NfcAdaptation::NFCA_TASK() ** ** Description: NFCA_TASK runs the GKI main task ** ** Returns:     none ** *******************************************************************************/ uint32_t NfcAdaptation::NFCA_TASK(__attribute__((unused)) uint32_t arg) {  const char* func = "NfcAdaptation::NFCA_TASK";  LOG(VERBOSE) << StringPrintf("%s: enter", func);  GKI_run(nullptr);  LOG(VERBOSE) << StringPrintf("%s: exit", func);  return 0; } /******************************************************************************* ** ** Function:    NfcAdaptation::Thread() ** ** Description: Creates work threads ** ** Returns:     none ** *******************************************************************************/ uint32_t NfcAdaptation::Thread(__attribute__((unused)) uint32_t arg) {  const char* func = "NfcAdaptation::Thread";  LOG(VERBOSE) << StringPrintf("%s: enter", func);   {    ThreadCondVar CondVar;    AutoThreadMutex guard(CondVar);    GKI_create_task((TASKPTR)nfc_task, NFC_TASK, (int8_t*)"NFC_TASK", nullptr, 0,                    (pthread_cond_t*)CondVar, (pthread_mutex_t*)CondVar);    CondVar.wait();  }   NfcAdaptation::GetInstance().signal();   GKI_exit_task(GKI_get_taskid());  LOG(VERBOSE) << StringPrintf("%s: exit", func);  return 0; } /******************************************************************************* ** ** Function:    NfcAdaptation::GetHalEntryFuncs() ** ** Description: Get the set of HAL entry points. ** ** Returns:     Functions pointers for HAL entry points. ** *******************************************************************************/ tHAL_NFC_ENTRY* NfcAdaptation::GetHalEntryFuncs() { return &mHalEntryFuncs; } /******************************************************************************* ** ** Function:    NfcAdaptation::InitializeHalDeviceContext ** ** Description: Check validity of current handle to the nfc HAL service ** ** Returns:     None. ** *******************************************************************************/ void NfcAdaptation::InitializeHalDeviceContext() {  const char* func = "NfcAdaptation::InitializeHalDeviceContext";   mHalEntryFuncs.initialize = HalInitialize;  mHalEntryFuncs.terminate = HalTerminate;  mHalEntryFuncs.open = HalOpen;  mHalEntryFuncs.close = HalClose;  mHalEntryFuncs.core_initialized = HalCoreInitialized;  mHalEntryFuncs.write = HalWrite;  mHalEntryFuncs.prediscover = HalPrediscover;  mHalEntryFuncs.control_granted = HalControlGranted;  mHalEntryFuncs.power_cycle = HalPowerCycle;  mHalEntryFuncs.get_max_ee = HalGetMaxNfcee;  LOG(INFO) << StringPrintf("%s: INfc::getService()", func);  mAidlHal = nullptr;  mHal = mHal_1_1 = mHal_1_2 = nullptr;  if (!use_aidl) {    mHal = mHal_1_1 = mHal_1_2 = INfcV1_2::getService();  }  if (!use_aidl && mHal_1_2 == nullptr) {    mHal = mHal_1_1 = INfcV1_1::getService();    if (mHal_1_1 == nullptr) {      mHal = INfc::getService();    }  }  if (mHal == nullptr) {    // Try get AIDL    ::ndk::SpAIBinder binder(        AServiceManager_waitForService(NFC_AIDL_HAL_SERVICE_NAME.c_str()));    mAidlHal = INfcAidl::fromBinder(binder);    if (mAidlHal != nullptr) {      use_aidl = true;      AIBinder_linkToDeath(mAidlHal->asBinder().get(), mDeathRecipient.get(),                           nullptr /* cookie */);      mHal = mHal_1_1 = mHal_1_2 = nullptr;      mAidlHal->getInterfaceVersion(&mAidlHalVer);      LOG(INFO) << StringPrintf("%s: INfcAidl::fromBinder returned ver(%d)",                                func, mAidlHalVer);      // TODO: Enforce VSR API level check later      // if (get_vsr_api_level() <= __ANDROID_API_V__) {      if (mAidlHalVer <= 1) {        sVndExtnsPresent = sNfcVendorExtn->Initialize(nullptr, mAidlHal);      }    }    LOG_ALWAYS_FATAL_IF(mAidlHal == nullptr,                        "Failed to retrieve the NFC AIDL!");  } else {    LOG(INFO) << StringPrintf("%s: INfc::getService() returned %p (%s)", func,                              mHal.get(),                              (mHal->isRemote() ? "remote" : "local"));    mNfcHalDeathRecipient = new NfcHalDeathRecipient(mHal);    mHal->linkToDeath(mNfcHalDeathRecipient, 0);    sVndExtnsPresent = sNfcVendorExtn->Initialize(mHal, nullptr);  } } /******************************************************************************* ** ** Function:    NfcAdaptation::HalInitialize ** ** Description: Not implemented because this function is only needed **              within the HAL. ** ** Returns:     None. ** *******************************************************************************/ void NfcAdaptation::HalInitialize() {  const char* func = "NfcAdaptation::HalInitialize";  LOG(VERBOSE) << StringPrintf("%s", func); } /******************************************************************************* ** ** Function:    NfcAdaptation::HalTerminate ** ** Description: Not implemented because this function is only needed **              within the HAL. ** ** Returns:     None. ** *******************************************************************************/ void NfcAdaptation::HalTerminate() {  const char* func = "NfcAdaptation::HalTerminate";  LOG(VERBOSE) << StringPrintf("%s", func); } /******************************************************************************* ** ** Function:    NfcAdaptation::HalOpenInternal ** ** Description: Turn on controller, download firmware. ** ** Returns:     None. ** *******************************************************************************/ void NfcAdaptation::HalOpenInternal(tHAL_NFC_CBACK* p_hal_cback,                                    tHAL_NFC_DATA_CBACK* p_data_cback) {  const char* func = "NfcAdaptation::HalOpenInternal";  LOG(VERBOSE) << StringPrintf("%s", func);  if (sVndExtnsPresent) {    sNfcVendorExtn->setNciCallback(p_hal_cback, p_data_cback);  }  if (mAidlHal != nullptr) {    mAidlCallback = ::ndk::SharedRefBase::make<NfcAidlClientCallback>(        p_hal_cback, p_data_cback);    Status status = mAidlHal->open(mAidlCallback);    if (!status.isOk()) {      LOG(ERROR) << StringPrintf(          "%s: Open Error=%s", __func__,          ::aidl::android::hardware::nfc::toString(              static_cast<NfcAidlStatus>(status.getServiceSpecificError()))              .c_str());    } else {      bool verbose_vendor_log =          android::base::GetBoolProperty(VERBOSE_VENDOR_LOG_PROPERTY, false);      mAidlHal->setEnableVerboseLogging(verbose_vendor_log);      LOG(VERBOSE) << StringPrintf("%s: verbose_vendor_log=%u", __func__,                                   verbose_vendor_log);    }  } else if (mHal_1_1 != nullptr) {    mCallback = new NfcClientCallback(p_hal_cback, p_data_cback);    mHal_1_1->open_1_1(mCallback);  } else if (mHal != nullptr) {    mCallback = new NfcClientCallback(p_hal_cback, p_data_cback);    mHal->open(mCallback);  } } /******************************************************************************* ** ** Function:    NfcAdaptation::HalOpen ** ** Description: Invoke HalOpenInternal in separate thread to not to block **              caller. ** ** Returns:     None. ** *******************************************************************************/ void NfcAdaptation::HalOpen(tHAL_NFC_CBACK* p_hal_cback,                            tHAL_NFC_DATA_CBACK* p_data_cback) {  const char* func = "NfcAdaptation::HalOpen";  LOG(VERBOSE) << StringPrintf("%s", func);  std::thread([p_hal_cback, p_data_cback]() {    HalOpenInternal(p_hal_cback, p_data_cback);  }).detach(); } /******************************************************************************* ** ** Function:    NfcAdaptation::HalClose ** ** Description: Turn off controller. ** ** Returns:     None. ** *******************************************************************************/ void NfcAdaptation::HalClose() {  const char* func = "NfcAdaptation::HalClose";  LOG(VERBOSE) << StringPrintf("%s", func);  if (sVndExtnsPresent) {    sNfcVendorExtn->processEvent(HANDLE_NFC_HAL_CLOSE, HAL_NFC_STATUS_OK);  }  if (mAidlHal != nullptr) {    mAidlHal->close(NfcCloseType::DISABLE);  } else if (mHal != nullptr) {    mHal->close();  } } /******************************************************************************* ** ** Function:    NfcAdaptation::HalWrite ** ** Description: Write NCI message to the controller. ** ** Returns:     None. ** *******************************************************************************/ void NfcAdaptation::HalWrite(uint16_t data_len, uint8_t* p_data) {  const char* func = "NfcAdaptation::HalWrite";  LOG(VERBOSE) << StringPrintf("%s", func);   if (sVndExtnsPresent) {    bool isVndExtSpecCmd = sNfcVendorExtn->processCmd(data_len, p_data);    // If true to be handled in extension, otherwise processed to hal    if (isVndExtSpecCmd) {      return;    }  }  if (mAidlHal != nullptr) {    int ret;    std::vector<uint8_t> aidl_data(p_data, p_data + data_len);    mAidlHal->write(aidl_data, &ret);  } else if (mHal != nullptr) {    ::android::hardware::nfc::V1_0::NfcData data;    data.setToExternal(p_data, data_len);    mHal->write(data);  } } /******************************************************************************* ** ** Function:    NfcAdaptation::HalCoreInitialized ** ** Description: Adjust the configurable parameters in the controller. ** ** Returns:     None. ** *******************************************************************************/ void NfcAdaptation::HalCoreInitialized(uint16_t data_len,                                       uint8_t* p_core_init_rsp_params) {  const char* func = "NfcAdaptation::HalCoreInitialized";  LOG(VERBOSE) << StringPrintf("%s", func);  if (sVndExtnsPresent) {    sNfcVendorExtn->processEvent(HANDLE_NFC_HAL_CORE_INITIALIZE,                                 HAL_NFC_STATUS_OK);  }  if (mAidlHal != nullptr) {    // AIDL coreInitialized doesn't send data to HAL.    mAidlHal->coreInitialized();  } else if (mHal != nullptr) {    hidl_vec<uint8_t> data;    data.setToExternal(p_core_init_rsp_params, data_len);    mHal->coreInitialized(data);  } } /******************************************************************************* ** ** Function:    NfcAdaptation::HalPrediscover ** ** Description:     Perform any vendor-specific pre-discovery actions (if **                  needed) If any actions were performed TRUE will be returned, **                  and HAL_PRE_DISCOVER_CPLT_EVT will notify when actions are **                  completed. ** ** Returns:         TRUE if vendor-specific pre-discovery actions initialized **                  FALSE if no vendor-specific pre-discovery actions are **                  needed. ** *******************************************************************************/ bool NfcAdaptation::HalPrediscover() {  const char* func = "NfcAdaptation::HalPrediscover";  LOG(VERBOSE) << StringPrintf("%s", func);  if (sVndExtnsPresent) {    sNfcVendorExtn->processEvent(HANDLE_NFC_PRE_DISCOVER, HAL_NFC_STATUS_OK);  }  if (mAidlHal != nullptr) {    Status status = mAidlHal->preDiscover();    if (status.isOk()) {      LOG(VERBOSE) << StringPrintf("%s: wait for NFC_PRE_DISCOVER_CPLT_EVT",                                   func);      return true;    }  } else if (mHal != nullptr) {    mHal->prediscover();  }   return false; } /******************************************************************************* ** ** Function:        HAL_NfcControlGranted ** ** Description:     Grant control to HAL control for sending NCI commands. **                  Call in response to HAL_REQUEST_CONTROL_EVT. **                  Must only be called when there are no NCI commands pending. **                  HAL_RELEASE_CONTROL_EVT will notify when HAL no longer **                  needs control of NCI. ** ** Returns:         void ** *******************************************************************************/ void NfcAdaptation::HalControlGranted() {  const char* func = "NfcAdaptation::HalControlGranted";  LOG(VERBOSE) << StringPrintf("%s", func);  if (mAidlHal != nullptr) {    if (mAidlHalVer > 1) {      NfcAidlStatus aidl_status;      mAidlHal->controlGranted(&aidl_status);    } else {      LOG(ERROR) << StringPrintf("%s: Unsupported function", func);    }  } else if (mHal != nullptr) {    mHal->controlGranted();  } } /******************************************************************************* ** ** Function:    NfcAdaptation::HalPowerCycle ** ** Description: Turn off and turn on the controller. ** ** Returns:     None. ** *******************************************************************************/ void NfcAdaptation::HalPowerCycle() {  const char* func = "NfcAdaptation::HalPowerCycle";  LOG(VERBOSE) << StringPrintf("%s", func);  if (sVndExtnsPresent) {    sNfcVendorExtn->processEvent(HANDLE_NFC_HAL_POWER_CYCLE, HAL_NFC_STATUS_OK);  }  if (mAidlHal != nullptr) {    mAidlHal->powerCycle();  } else if (mHal != nullptr) {    mHal->powerCycle();  } } /******************************************************************************* ** ** Function:    NfcAdaptation::HalGetMaxNfcee ** ** Description: Turn off and turn on the controller. ** ** Returns:     None. ** *******************************************************************************/ uint8_t NfcAdaptation::HalGetMaxNfcee() {  const char* func = "NfcAdaptation::HalGetMaxNfcee";  LOG(VERBOSE) << StringPrintf("%s", func);  if (sVndExtnsPresent) {    sNfcVendorExtn->processEvent(HANDLE_NFC_GET_MAX_NFCEE, HAL_NFC_STATUS_OK);  }  return nfa_ee_max_ee_cfg; } /******************************************************************************* ** ** Function:    NfcAdaptation::DownloadFirmware ** ** Description: Download firmware patch files. ** ** Returns:     None. ** *******************************************************************************/ bool NfcAdaptation::DownloadFirmware() {  const char* func = "NfcAdaptation::DownloadFirmware";  isDownloadFirmwareCompleted = false;  LOG(VERBOSE) << StringPrintf("%s: enter", func);  HalInitialize();  if (sVndExtnsPresent) {    sNfcVendorExtn->processEvent(HANDLE_DOWNLOAD_FIRMWARE_REQUEST,                                 HAL_NFC_STATUS_OK);  }  mHalOpenCompletedEvent.lock();  LOG(VERBOSE) << StringPrintf("%s: try open HAL", func);  HalOpen(HalDownloadFirmwareCallback, HalDownloadFirmwareDataCallback);  // wait up to 60s, if not opened in this delay, give up and close anyway.  mHalOpenCompletedEvent.wait(60000);  mHalOpenCompletedEvent.unlock();   LOG(VERBOSE) << StringPrintf("%s: try core init HAL", func);  uint8_t coreInitRspParams = 0;  HalCoreInitialized(sizeof(uint8_t), &coreInitRspParams);   LOG(VERBOSE) << StringPrintf("%s: try close HAL", func);  HalClose();   HalTerminate();  LOG(VERBOSE) << StringPrintf("%s: exit", func);   return isDownloadFirmwareCompleted; } /******************************************************************************* ** ** Function:    NfcAdaptation::HalDownloadFirmwareCallback ** ** Description: Receive events from the HAL. ** ** Returns:     None. ** *******************************************************************************/ void NfcAdaptation::HalDownloadFirmwareCallback(nfc_event_t event,                                                nfc_status_t event_status) {  const char* func = "NfcAdaptation::HalDownloadFirmwareCallback";  LOG(VERBOSE) << StringPrintf("%s: event=0x%X", func, event);  switch (event) {    case HAL_NFC_OPEN_CPLT_EVT: {      LOG(VERBOSE) << StringPrintf("%s: HAL_NFC_OPEN_CPLT_EVT", func);      if (event_status == HAL_NFC_STATUS_OK) isDownloadFirmwareCompleted = true;      mHalOpenCompletedEvent.signal();      break;    }    case HAL_NFC_CLOSE_CPLT_EVT: {      LOG(VERBOSE) << StringPrintf("%s: HAL_NFC_CLOSE_CPLT_EVT", func);      break;    }  }  tNFC_HAL_EVT_MSG* p_msg =      (tNFC_HAL_EVT_MSG*)GKI_getbuf(sizeof(tNFC_HAL_EVT_MSG));  if (p_msg != nullptr) {    /* Initialize NFC_HDR */    p_msg->hdr.len = 0;    p_msg->hdr.event = BT_EVT_TO_NFC_MSGS;    p_msg->hdr.offset = 0;    p_msg->hdr.layer_specific = 0;    p_msg->hal_evt = event;    p_msg->status = event_status;    GKI_send_msg(NFC_TASK, NFC_MBOX_ID, p_msg);  } else {    LOG(ERROR) << StringPrintf("No buffer");  }; } /******************************************************************************* ** ** Function:    NfcAdaptation::HalDownloadFirmwareDataCallback ** ** Description: Receive data events from the HAL. ** ** Returns:     None. ** *******************************************************************************/ void NfcAdaptation::HalDownloadFirmwareDataCallback(uint16_t data_len,                                                    uint8_t* p_data) {  const char* func = "NfcAdaptation::HalDownloadFirmwareDataCallback";  LOG(VERBOSE) << StringPrintf("%s: data_len= %d", func, data_len);  if (p_data == nullptr) {    LOG(ERROR) << StringPrintf("%s: Invalid data!", func);    return;  }  NFC_HDR* p_msg = (NFC_HDR*)GKI_getbuf(sizeof(NFC_HDR) +                                        NFC_RECEIVE_MSGS_OFFSET + data_len);  if (p_msg != nullptr) {    /* Initialize NFC_HDR */    p_msg->len = data_len;    p_msg->event = BT_EVT_TO_NFC_NCI;    p_msg->offset = NFC_RECEIVE_MSGS_OFFSET;     /* no need to check length, it always less than pool size */    memcpy((uint8_t*)(p_msg + 1) + p_msg->offset, p_data, p_msg->len);     GKI_send_msg(NFC_TASK, NFC_MBOX_ID, p_msg);    LOG(VERBOSE) << StringPrintf("GKI msg sent!");  } else {    LOG(ERROR) << StringPrintf("No buffer");  } } /******************************************************************************* ** ** Function:    ThreadMutex::ThreadMutex() ** ** Description: class constructor ** ** Returns:     none ** *******************************************************************************/ ThreadMutex::ThreadMutex() {  pthread_mutexattr_t mutexAttr;   pthread_mutexattr_init(&mutexAttr);  pthread_mutexattr_settype(&mutexAttr, PTHREAD_MUTEX_RECURSIVE);  pthread_mutex_init(&mMutex, &mutexAttr);  pthread_mutexattr_destroy(&mutexAttr); } /******************************************************************************* ** ** Function:    ThreadMutex::~ThreadMutex() ** ** Description: class destructor ** ** Returns:     none ** *******************************************************************************/ ThreadMutex::~ThreadMutex() { pthread_mutex_destroy(&mMutex); } /******************************************************************************* ** ** Function:    ThreadMutex::lock() ** ** Description: lock kthe mutex ** ** Returns:     none ** *******************************************************************************/ void ThreadMutex::lock() { pthread_mutex_lock(&mMutex); } /******************************************************************************* ** ** Function:    ThreadMutex::unblock() ** ** Description: unlock the mutex ** ** Returns:     none ** *******************************************************************************/ void ThreadMutex::unlock() { pthread_mutex_unlock(&mMutex); } /******************************************************************************* ** ** Function:    ThreadCondVar::ThreadCondVar() ** ** Description: class constructor ** ** Returns:     none ** *******************************************************************************/ ThreadCondVar::ThreadCondVar() {  pthread_condattr_t CondAttr;   pthread_condattr_init(&CondAttr);  pthread_condattr_setclock(&CondAttr, CLOCK_MONOTONIC);  pthread_cond_init(&mCondVar, &CondAttr);   pthread_condattr_destroy(&CondAttr); } /******************************************************************************* ** ** Function:    ThreadCondVar::~ThreadCondVar() ** ** Description: class destructor ** ** Returns:     none ** *******************************************************************************/ ThreadCondVar::~ThreadCondVar() { pthread_cond_destroy(&mCondVar); } /******************************************************************************* ** ** Function:    ThreadCondVar::wait() ** ** Description: wait on the mCondVar ** ** Returns:     none ** *******************************************************************************/ void ThreadCondVar::wait() {  pthread_cond_wait(&mCondVar, *this);  pthread_mutex_unlock(*this); } /******************************************************************************* ** ** Function:    ThreadCondVar::wait() ** ** Description: wait on the mCondVar for the indicated amount of time (ms) ** ** Returns:     true if wait succeeded ** *******************************************************************************/ bool ThreadCondVar::wait(long millisec) {  bool retVal = false;  struct timespec absoluteTime;   if (clock_gettime(CLOCK_MONOTONIC, &absoluteTime) == -1) {    LOG(ERROR) << StringPrintf("ThreadCondVar::wait: fail get time; errno=0x%X",                               errno);  } else {    absoluteTime.tv_sec += millisec / 1000;    long ns = absoluteTime.tv_nsec + ((millisec % 1000) * 1000000);    if (ns > 1000000000) {      absoluteTime.tv_sec++;      absoluteTime.tv_nsec = ns - 1000000000;    } else      absoluteTime.tv_nsec = ns;  }   int waitResult = pthread_cond_timedwait(&mCondVar, *this, &absoluteTime);  if ((waitResult != 0) && (waitResult != ETIMEDOUT))    LOG(ERROR) << StringPrintf("ThreadCondVar::wait: fail timed wait; error=0x%X",                               waitResult);  retVal = (waitResult == 0);  // waited successfully  if (retVal) pthread_mutex_unlock(*this);  return retVal; } /******************************************************************************* ** ** Function:    ThreadCondVar::signal() ** ** Description: signal the mCondVar ** ** Returns:     none ** *******************************************************************************/ void ThreadCondVar::signal() {  AutoThreadMutex a(*this);  pthread_cond_signal(&mCondVar); } /******************************************************************************* ** ** Function:    AutoThreadMutex::AutoThreadMutex() ** ** Description: class constructor, automatically lock the mutex ** ** Returns:     none ** *******************************************************************************/ AutoThreadMutex::AutoThreadMutex(ThreadMutex& m) : mm(m) { mm.lock(); } /******************************************************************************* ** ** Function:    AutoThreadMutex::~AutoThreadMutex() ** ** Description: class destructor, automatically unlock the mutex ** ** Returns:     none ** *******************************************************************************/ AutoThreadMutex::~AutoThreadMutex() { mm.unlock(); } 根据nfcadaption，主要介绍其功能和定位









已深度思考（用时 52 秒）



根据提供的代码，NfcAdaptation 是 Android NFC 系统中的一个适配层，主要用于连接上层 NFC 框架和底层的 HAL（硬件抽象层）。它负责初始化、配置和与 HAL 交互，同时处理一些供应商扩展功能。以下是其主要功能和定位的详细介绍：

\### 1. **功能概述**

   \- **初始化与终止**：负责初始化 NFC 硬件、配置参数，并在结束时进行清理。

   \- **与 HAL 通信**：提供与 HAL 的接口，包括打开、关闭、读写操作等。

   \- **事件处理**：处理来自 HAL 的事件（如打开完成、关闭完成等）和数据回调。

   \- **供应商扩展支持**：支持供应商特定的扩展功能。

   \- **配置管理**：从配置文件中读取并设置 NFC 参数。

   \- **多线程管理**：使用线程和互斥锁来管理并发操作。

   \- **固件下载**：提供下载 NFC 控制器固件的功能。

   \- **日志与调试**：支持调试日志和崩溃日志的存储。

\### 2. **关键组件**

   \- **单例模式**：通过 `GetInstance` 方法获取唯一的实例。

   \- **HAL 接口**：支持 AIDL（Android Interface Definition Language）和 HIDL（Hardware Interface Definition Language）两种 HAL 接口。

   \- **回调机制**：通过 `tHAL_NFC_CBACK` 和 `tHAL_NFC_DATA_CBACK` 处理 HAL 事件和数据。

   \- **线程同步**：使用 `ThreadMutex` 和 `ThreadCondVar` 实现线程同步。

   \- **供应商扩展**：通过 `NfcVendorExtn` 类处理供应商特定的逻辑。

\### 3. **主要方法**

   \- **Initialize()**: 初始化 NFC 适配层，包括读取配置、初始化日志、创建任务等。

   \- **Finalize()**: 清理资源，关闭 HAL 连接。

   \- **HalOpen()**: 打开 HAL 连接，并注册回调函数。

   \- **HalWrite()**: 向 HAL 写入数据（NCI 命令）。

   \- **HalCoreInitialized()**: 在核心初始化后调用，用于配置控制器参数。

   \- **DownloadFirmware()**: 下载固件到 NFC 控制器。

   \- **GetHalEntryFuncs()**: 获取 HAL 入口函数结构体，供上层调用。

\### 4. **与 HAL 的交互**

   \- 通过 `mAidlHal`（AIDL 接口）或 `mHal`（HIDL 接口）与底层 HAL 通信。

   \- 支持不同版本的 HAL（1.0、1.1、1.2），并自动选择可用的版本。

   \- 处理 HAL 的死亡通知（binder died），在 HAL 崩溃时重启服务。

\### 5. **供应商扩展**

   \- 通过 `sNfcVendorExtn` 实例处理供应商特定的命令和事件。

   \- 在关键流程中（如打开、关闭、读写）调用供应商扩展的处理函数。

\### 6. **配置管理**

   \- 从 `NfcConfig` 中读取配置参数（如日志级别、NCI 重置类型等）。

   \- 将配置参数传递给 HAL（通过 `getConfig` 方法）。

\### 7. **多线程**

   \- 使用 GKI（Generic Kernel Interface）任务管理。

   \- 创建独立的线程处理 NFC 任务（如 `NFCA_TASK` 和 `Thread`）。

\### 8. **日志与调试**

   \- 支持详细的日志输出（通过 `android-base/logging`）。

   \- 在 HAL 崩溃时存储日志到指定路径（`DEFAULT_CRASH_LOGS_PATH`）。

\### 9. **固件下载流程**

   \- 通过 `DownloadFirmware` 方法下载固件。

   \- 使用 `HalDownloadFirmwareCallback` 和 `HalDownloadFirmwareDataCallback` 处理下载过程中的事件和数据。

\### 10. **定位**

   \- `NfcAdaptation` 是 Android NFC 栈中的中间层，位于上层 NFC 框架（如 `NfcService`）和底层 HAL 之间。

   \- 它封装了与硬件相关的操作，使上层可以独立于具体的硬件实现。

   \- 同时，它提供了扩展点（供应商扩展）以支持不同厂商的定制功能。

\### 总结

`NfcAdaptation` 是 Android NFC 系统中的核心适配层，负责桥接上层应用和底层硬件。它处理初始化、配置、通信、事件处理、供应商扩展等关键任务，确保 NFC 功能的稳定运行。通过支持 AIDL 和 HIDL 两种接口，它能够兼容不同版本的 Android 系统和硬件实现。

### NfcAdaptation 功能与定位

`NfcAdaptation` 是 Android NFC 协议栈中的 **硬件抽象层适配器**，核心职责是桥接上层 NFC 框架（如 `NfcService`）与底层硬件驱动（HAL）。它管理 HAL 的初始化、通信、事件处理及平台适配逻辑，确保 NFC 功能跨硬件平台的一致性。

------

### 主要功能详解

#### 1. **HAL 接口管理**

- **动态绑定 HAL 服务**：

  - 自动选择 AIDL（新架构）或 HIDL（旧架构）接口。
  - 支持 HAL 版本协商（1.0/1.1/1.2），优先使用高版本。

  cpp

  

  复制

  

  下载

  ```
  if (!use_aidl) {
    mHal = mHal_1_1 = mHal_1_2 = INfcV1_2::getService();
    if (!mHal_1_2) mHal = mHal_1_1 = INfcV1_1::getService();
    // ... 回退逻辑
  }
  ```

- **死亡监听**：

  - 监控 HAL 进程崩溃，触发重启或日志保存。

  cpp

  

  复制

  

  下载

  ```
  AIBinder_linkToDeath(mAidlHal->asBinder(), mDeathRecipient.get(), nullptr);
  ```

#### 2. **配置管理**

- **解析系统配置**：

  - 从 `NfcConfig` 读取参数（如日志级别、路由规则）。

  cpp

  

  复制

  

  下载

  ```
  nfc_storage_path = NfcConfig::getString(NAME_NFA_STORAGE, "/data/nfc");
  ```

- **同步 HAL 配置**：

  - 获取 HAL 的厂商配置（`getConfig()`），并映射到协议栈变量。

  cpp

  

  复制

  

  下载

  ```
  void GetVendorConfigs(std::map<std::string, ConfigValue>& configMap) {
    if (mAidlHal) mAidlHal->getConfig(&aidlConfigValue);
    // ... 填充 configMap
  }
  ```

#### 3. **数据通信**

- **命令转发**：

  - 将上层 NCI 命令转发给 HAL。

  cpp

  

  复制

  

  下载

  ```
  void HalWrite(uint16_t len, uint8_t* data) {
    if (mAidlHal) {
      std::vector<uint8_t> aidl_data(data, data + len);
      mAidlHal->write(aidl_data, &ret);
    }
  }
  ```

- **事件回调处理**：

  - 处理 HAL 事件（如 `HAL_NFC_OPEN_CPLT_EVT`）并通知上层。

  cpp

  

  复制

  

  下载

  ```
  Return<void> sendEvent(NfcEvent event, NfcStatus status) override {
    mEventCallback((uint8_t)event, (tHAL_NFC_STATUS)status);
  }
  ```

#### 4. **生命周期管理**

- **初始化/终止**：

  - 启动/停止 GKI 任务线程，管理非易失存储。

  cpp

  

  复制

  

  下载

  ```
  void Initialize() {
    GKI_init();
    GKI_create_task((TASKPTR)NFCA_TASK, ...);
  }
  ```

- **固件下载**：

  - 独立流程下载 NFC 控制器固件。

  cpp

  

  复制

  

  下载

  ```
  bool DownloadFirmware() {
    HalOpen(HalDownloadFirmwareCallback, ...);
    // ... 阻塞等待完成
  }
  ```

#### 5. **平台适配扩展**

- **供应商扩展支持**：

  - 通过 `NfcVendorExtn` 处理厂商定制逻辑。

  cpp

  

  复制

  

  下载

  ```
  if (sVndExtnsPresent) {
    sNfcVendorExtn->processCmd(data_len, p_data); // 拦截定制命令
  }
  ```

- **兼容性处理**：

  - 根据 Android 版本选择接口（如 VSR API Level 检查）。

------

### 关键定位

1. **抽象层**
   封装硬件差异，提供统一接口给上层 NFC 协议栈（如 `nfc_task`）。
2. **桥接器**
   转换框架请求为 HAL 调用，并处理异步事件回调。
3. **扩展点**
   通过 `NfcVendorExtn` 支持 OEM 厂商定制功能（如特殊指令处理）。
4. **生命周期协调者**
   管理 NFC 控制器的启动、配置、关闭及异常恢复流程。

------

### 典型流程

图表

代码







下载

HALNfcAdaptationNfcServiceHALNfcAdaptationNfcServiceInitialize()getConfig()返回配置应用配置HalOpen()open()HAL_NFC_OPEN_CPLT_EVT通知打开完成HalWrite(NCI 命令)write()数据回调转发响应

------

### 设计亮点

- **多 HAL 版本兼容**：无缝切换 AIDL/HIDL。
- **线程安全**：通过 `ThreadMutex` 和 `ThreadCondVar` 同步关键操作。
- **厂商扩展机制**：通过 `sVndExtnsPresent` 标志动态启用定制逻辑。
- **崩溃恢复**：HAL 崩溃时保存日志并重启进程。

通过这一适配层，Android NFC 实现了硬件无关性，确保不同厂商的 NFC 芯片均可通过统一接口集成。









GKI_init();  GKI_enable();  GKI_create_task((TASKPTR)NFCA_TASK, BTU_TASK, (int8_t*)"NFCA_TASK", nullptr, 0,                  (pthread_cond_t*)nullptr, nullptr); 作用









已深度思考（用时 27 秒）



这段代码是用于初始化和管理任务调度的，特别是在NFC协议栈中。我们来分解一下：

1. `GKI_init();`  

   \- 这是GKI（Generic Kernel Interface）的初始化函数。GKI是Android NFC协议栈中的一个抽象层，用于提供任务调度、缓冲区管理、定时器等功能，使得上层代码可以在不同的操作系统上运行。

   \- 它初始化GKI模块的内部数据结构，为后续操作做准备。

2. `GKI_enable();`  

   \- 这个函数启用GKI调度器。在调用此函数之后，GKI将开始调度任务（即运行在GKI环境中的任务）。通常，在初始化GKI并创建了必要的任务之后，会调用此函数来启动任务调度。

3. `GKI_create_task(...);`  

   \- 这个函数创建一个新的任务（线程）并加入到GKI的调度中。

   \- 参数解释：

​     \- `(TASKPTR)NFCA_TASK`: 这是任务的主函数指针，即任务启动后将执行`NFCA_TASK`函数。

​     \- `BTU_TASK`: 这是任务的ID（在GKI内部使用，用于标识任务）。

​     \- `(int8_t*)"NFCA_TASK"`: 任务的名称（字符串），用于调试。

​     \- `nullptr`: 传递给任务函数的参数（这里没有参数）。

​     \- `0`: 堆栈大小（0表示使用默认大小）。

​     \- `(pthread_cond_t*)nullptr`: 条件变量指针（这里不使用，所以为nullptr）。

​     \- `nullptr`: 互斥锁指针（这里也不使用）。

   因此，这行代码创建了一个名为"NFCA_TASK"的任务，该任务将运行`NFCA_TASK`函数，任务ID为`BTU_TASK`。

\### 整体作用：

这三行代码一起完成了GKI环境的初始化，并创建了一个任务（线程）来运行NFC协议栈中的某个功能（具体功能由`NFCA_TASK`函数定义）。然后，通过`GKI_enable()`启动调度，使得这个任务可以运行。

在NFC协议栈中，通常会有多个任务，例如处理NCI命令的任务、处理数据交换的任务等。这里创建的任务`NFCA_TASK`可能是其中一个重要任务，负责NFC适配层（Adaptation Layer）的主要工作。

注意：`GKI`是Android NFC协议栈中一个较老的调度模块，现在新的代码可能已经使用其他方式（如直接使用pthread）来管理线程，但在一些旧代码中仍然可以看到。

这段代码是 Android NFC 协议栈中**通用内核接口(GKI)**的初始化操作，用于创建和管理 NFC 的核心任务线程。各函数作用如下：

cpp



复制



下载

```
GKI_init();                     // 1. 初始化GKI内核
GKI_enable();                   // 2. 启用任务调度器
GKI_create_task(                // 3. 创建NFCA核心任务
    (TASKPTR)NFCA_TASK,         //    - 任务入口函数
    BTU_TASK,                   //    - 任务ID (蓝牙协议栈任务ID复用)
    (int8_t*)"NFCA_TASK",       //    - 任务名称 (调试用)
    nullptr,                    //    - 任务参数 (无)
    0,                          //    - 栈大小 (默认)
    (pthread_cond_t*)nullptr,   //    - 条件变量 (无)
    nullptr);                   //    - 互斥锁 (无)
```

------

### 具体作用解析：

1. **`GKI_init()`**

   - 初始化 GKI 内核数据结构
   - 创建内存池/缓冲区管理系统
   - 初始化定时器和任务队列
   - 准备多任务调度环境

2. **`GKI_enable()`**

   - 激活任务调度器
   - 允许后续创建的线程被调度执行
   - 启动内核的事件处理循环

3. **`GKI_create_task()`**

   | 参数                 | 说明                           |
   | :------------------- | :----------------------------- |
   | `(TASKPTR)NFCA_TASK` | NFC 主任务函数指针             |
   | `BTU_TASK`           | 任务 ID (复用蓝牙协议栈任务ID) |
   | `"NFCA_TASK"`        | 任务名称 (调试可见)            |
   | `nullptr`            | 无参数传递给任务               |
   | `0`                  | 使用默认栈大小 (通常 4-8KB)    |

   → 创建名为 "NFCA_TASK" 的线程，执行 `NFCA_TASK()` 函数

------

### 整体执行流程：

图表

代码







下载

NFCA_TASK_ThreadGKI_KernelMainThreadNFCA_TASK_ThreadGKI_KernelMainThreadloop[任务循环]GKI_init()初始化内存池/定时器/队列GKI_enable()激活调度器GKI_create_task()创建线程执行NFCA_TASK()处理NFC事件返回待处理事件

------

### NFC 核心任务的作用 (`NFCA_TASK()`)：

cpp



复制



下载

```
uint32_t NFCA_TASK(uint32_t arg) {
  GKI_run(nullptr);  // 启动GKI主循环
  return 0;
}
```

- **`GKI_run()`** 内部实现：

  cpp

  

  复制

  

  下载

  ```
  while(true) {
    event = GKI_wait();     // 等待事件
    GKI_dispatch(event);    // 分发到处理函数
  }
  ```

  负责：

  1. 处理 NFC 数据帧的收发
  2. 调度定时器超时事件
  3. 协调各子模块（NCI/RW/CE/EE）的状态机
  4. 管理协议栈与 HAL 层的交互

------

### 架构定位：

图表

代码







下载

<svg role="graphics-document document" viewBox="-3.5 -3.5 1127.65283203125 77" class="flowchart mermaid-svg" xmlns="http://www.w3.org/2000/svg" width="100%" id="mermaid-svg-27" height="100%" style="max-width: 100%; transform-origin: 0px 0px; user-select: none; transform: translate(0px, 74.3264px) scale(1);"><g><marker orient="auto" markerHeight="8" markerWidth="8" markerUnits="userSpaceOnUse" refY="5" refX="5" viewBox="0 0 10 10" class="marker flowchart-v2" id="mermaid-svg-27_flowchart-v2-pointEnd"><path style="stroke-width: 1; stroke-dasharray: 1, 0;" class="arrowMarkerPath" d="M 0 0 L 10 5 L 0 10 z"></path></marker><marker orient="auto" markerHeight="8" markerWidth="8" markerUnits="userSpaceOnUse" refY="5" refX="4.5" viewBox="0 0 10 10" class="marker flowchart-v2" id="mermaid-svg-27_flowchart-v2-pointStart"><path style="stroke-width: 1; stroke-dasharray: 1, 0;" class="arrowMarkerPath" d="M 0 5 L 10 10 L 10 0 z"></path></marker><marker orient="auto" markerHeight="11" markerWidth="11" markerUnits="userSpaceOnUse" refY="5" refX="11" viewBox="0 0 10 10" class="marker flowchart-v2" id="mermaid-svg-27_flowchart-v2-circleEnd"><circle style="stroke-width: 1; stroke-dasharray: 1, 0;" class="arrowMarkerPath" r="5" cy="5" cx="5"></circle></marker><marker orient="auto" markerHeight="11" markerWidth="11" markerUnits="userSpaceOnUse" refY="5" refX="-1" viewBox="0 0 10 10" class="marker flowchart-v2" id="mermaid-svg-27_flowchart-v2-circleStart"><circle style="stroke-width: 1; stroke-dasharray: 1, 0;" class="arrowMarkerPath" r="5" cy="5" cx="5"></circle></marker><marker orient="auto" markerHeight="11" markerWidth="11" markerUnits="userSpaceOnUse" refY="5.2" refX="12" viewBox="0 0 11 11" class="marker cross flowchart-v2" id="mermaid-svg-27_flowchart-v2-crossEnd"><path style="stroke-width: 2; stroke-dasharray: 1, 0;" class="arrowMarkerPath" d="M 1,1 l 9,9 M 10,1 l -9,9"></path></marker><marker orient="auto" markerHeight="11" markerWidth="11" markerUnits="userSpaceOnUse" refY="5.2" refX="-1" viewBox="0 0 11 11" class="marker cross flowchart-v2" id="mermaid-svg-27_flowchart-v2-crossStart"><path style="stroke-width: 2; stroke-dasharray: 1, 0;" class="arrowMarkerPath" d="M 1,1 l 9,9 M 10,1 l -9,9"></path></marker><g class="root"><g class="clusters"></g><g class="edgePaths"><path marker-end="url(#mermaid-svg-27_flowchart-v2-pointEnd)" style="" class="edge-thickness-normal edge-pattern-solid edge-thickness-normal edge-pattern-solid flowchart-link" id="L_NfcService_libnfc-nci_0" d="M143.736,35L149.76,35C155.785,35,167.833,35,179.215,35C190.597,35,201.313,35,206.67,35L212.028,35"></path><path marker-end="url(#mermaid-svg-27_flowchart-v2-pointEnd)" style="" class="edge-thickness-normal edge-pattern-solid edge-thickness-normal edge-pattern-solid flowchart-link" id="L_libnfc-nci_NFCA_TASK_0" d="M352.208,35L361.082,35C369.956,35,387.704,35,404.785,35C421.866,35,438.28,35,446.487,35L454.694,35"></path><path marker-end="url(#mermaid-svg-27_flowchart-v2-pointEnd)" style="" class="edge-thickness-normal edge-pattern-solid edge-thickness-normal edge-pattern-solid flowchart-link" id="L_NFCA_TASK_NfcAdaptation_0" d="M598.792,35L607.96,35C617.127,35,635.463,35,653.132,35C670.801,35,687.803,35,696.304,35L704.806,35"></path><path marker-end="url(#mermaid-svg-27_flowchart-v2-pointEnd)" style="" class="edge-thickness-normal edge-pattern-solid edge-thickness-normal edge-pattern-solid flowchart-link" id="L_NfcAdaptation_NFC_HAL_0" d="M871.514,35L877.594,35C883.674,35,895.833,35,907.326,35C918.819,35,929.646,35,935.059,35L940.472,35"></path></g><g class="edgeLabels"><g transform="translate(179.88194751739502, 35)" class="edgeLabel"><g transform="translate(-11.145833015441895, -12)" class="label"><foreignObject height="24" width="22.29166603088379"><div class="labelBkg" xmlns="http://www.w3.org/1999/xhtml" style="background-color: rgba(88, 88, 88, 0.5); display: table-cell; white-space: nowrap; line-height: 1.5; max-width: 200px; text-align: center;"><span class="edgeLabel" style="fill: rgb(204, 204, 204); color: rgb(204, 204, 204); background-color: rgb(88, 88, 88); text-align: center;"><p style="margin: 0px; background-color: rgb(88, 88, 88);">JNI</p></span></div></foreignObject></g></g><g transform="translate(405.4513931274414, 35)" class="edgeLabel"><g transform="translate(-28.24305534362793, -12)" class="label"><foreignObject height="24" width="56.48611068725586"><div class="labelBkg" xmlns="http://www.w3.org/1999/xhtml" style="background-color: rgba(88, 88, 88, 0.5); display: table-cell; white-space: nowrap; line-height: 1.5; max-width: 200px; text-align: center;"><span class="edgeLabel" style="fill: rgb(204, 204, 204); color: rgb(204, 204, 204); background-color: rgb(88, 88, 88); text-align: center;"><p style="margin: 0px; background-color: rgb(88, 88, 88);">GKI调度</p></span></div></foreignObject></g></g><g transform="translate(653.7986221313477, 35)" class="edgeLabel"><g transform="translate(-30.00694465637207, -12)" class="label"><foreignObject height="24" width="60.01388931274414"><div class="labelBkg" xmlns="http://www.w3.org/1999/xhtml" style="background-color: rgba(88, 88, 88, 0.5); display: table-cell; white-space: nowrap; line-height: 1.5; max-width: 200px; text-align: center;"><span class="edgeLabel" style="fill: rgb(204, 204, 204); color: rgb(204, 204, 204); background-color: rgb(88, 88, 88); text-align: center;"><p style="margin: 0px; background-color: rgb(88, 88, 88);">HAL接口</p></span></div></foreignObject></g></g><g transform="translate(907.9930620193481, 35)" class="edgeLabel"><g transform="translate(-11.479166984558105, -12)" class="label"><foreignObject height="24" width="22.95833396911621"><div class="labelBkg" xmlns="http://www.w3.org/1999/xhtml" style="background-color: rgba(88, 88, 88, 0.5); display: table-cell; white-space: nowrap; line-height: 1.5; max-width: 200px; text-align: center;"><span class="edgeLabel" style="fill: rgb(204, 204, 204); color: rgb(204, 204, 204); background-color: rgb(88, 88, 88); text-align: center;"><p style="margin: 0px; background-color: rgb(88, 88, 88);">IPC</p></span></div></foreignObject></g></g></g><g class="nodes"><g transform="translate(75.86805725097656, 35)" id="flowchart-NfcService-0" class="node default"><rect height="54" width="135.73611450195312" y="-27" x="-67.86805725097656" style="" class="basic label-container"></rect><g transform="translate(-37.86805725097656, -12)" style="" class="label"><rect></rect><foreignObject height="24" width="75.73611450195312"><div xmlns="http://www.w3.org/1999/xhtml" style="display: table-cell; white-space: nowrap; line-height: 1.5; max-width: 200px; text-align: center;"><span class="nodeLabel" style="fill: rgb(204, 204, 204); color: rgb(204, 204, 204);"><p style="margin: 0px;">NfcService</p></span></div></foreignObject></g></g><g transform="translate(284.1180591583252, 35)" id="flowchart-libnfc-nci-1" class="node default"><rect height="54" width="136.18055725097656" y="-27" x="-68.09027862548828" style="" class="basic label-container"></rect><g transform="translate(-38.09027862548828, -12)" style="" class="label"><rect></rect><foreignObject height="24" width="76.18055725097656"><div xmlns="http://www.w3.org/1999/xhtml" style="display: table-cell; white-space: nowrap; line-height: 1.5; max-width: 200px; text-align: center;"><span class="nodeLabel" style="fill: rgb(204, 204, 204); color: rgb(204, 204, 204);"><p style="margin: 0px;">NFC协议栈</p></span></div></foreignObject></g></g><g transform="translate(528.7430629730225, 35)" id="flowchart-NFCA_TASK-3" class="node default"><rect height="54" width="140.09722137451172" y="-27" x="-70.04861068725586" style="" class="basic label-container"></rect><g transform="translate(-40.04861068725586, -12)" style="" class="label"><rect></rect><foreignObject height="24" width="80.09722137451172"><div xmlns="http://www.w3.org/1999/xhtml" style="display: table-cell; white-space: nowrap; line-height: 1.5; max-width: 200px; text-align: center;"><span class="nodeLabel" style="fill: rgb(204, 204, 204); color: rgb(204, 204, 204);"><p style="margin: 0px;">NFCA_TASK</p></span></div></foreignObject></g></g><g transform="translate(790.1597309112549, 35)" id="flowchart-NfcAdaptation-5" class="node default"><rect height="54" width="162.70833587646484" y="-27" x="-81.35416793823242" style="" class="basic label-container"></rect><g transform="translate(-51.35416793823242, -12)" style="" class="label"><rect></rect><foreignObject height="24" width="102.70833587646484"><div xmlns="http://www.w3.org/1999/xhtml" style="display: table-cell; white-space: nowrap; line-height: 1.5; max-width: 200px; text-align: center;"><span class="nodeLabel" style="fill: rgb(204, 204, 204); color: rgb(204, 204, 204);"><p style="margin: 0px;">NfcAdaptation</p></span></div></foreignObject></g></g><g transform="translate(1028.5625076293945, 35)" id="flowchart-NFC_HAL-7" class="node default"><rect height="54" width="168.18055725097656" y="-27" x="-84.09027862548828" style="" class="basic label-container"></rect><g transform="translate(-54.09027862548828, -12)" style="" class="label"><rect></rect><foreignObject height="24" width="108.18055725097656"><div xmlns="http://www.w3.org/1999/xhtml" style="display: table-cell; white-space: nowrap; line-height: 1.5; max-width: 200px; text-align: center;"><span class="nodeLabel" style="fill: rgb(204, 204, 204); color: rgb(204, 204, 204);"><p style="margin: 0px;">NFC硬件抽象层</p></span></div></foreignObject></g></g></g></g></g></svg>

此初始化流程构建了 NFC 协议栈的**执行引擎**，后续所有 NFC 操作（设备发现、数据交换、HCE处理等）都在此任务线程中调度完成。