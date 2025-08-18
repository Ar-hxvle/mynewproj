/****************************************************************************** * *  Copyright (C) 2010-2014 Broadcom Corporation * *  Licensed under the Apache License, Version 2.0 (the "License"); *  you may not use this file except in compliance with the License. *  You may obtain a copy of the License at: * *  http://www.apache.org/licenses/LICENSE-2.0 * *  Unless required by applicable law or agreed to in writing, software *  distributed under the License is distributed on an "AS IS" BASIS, *  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. *  See the License for the specific language governing permissions and *  limitations under the License. * ******************************************************************************/ /****************************************************************************** * *  NFA interface for device management * ******************************************************************************/ #include <android-base/logging.h> #include <android-base/stringprintf.h> #include <log/log.h> #include <string.h> #include "ndef_utils.h" #include "nfa_api.h" #include "nfa_ce_int.h" #include "nfa_nfcee_int.h" #include "nfa_wlc_int.h" #include "nfc_int.h" using android::base::StringPrintf; /***************************************************************************** **  Constants *****************************************************************************/ /***************************************************************************** **  APIs *****************************************************************************/ /******************************************************************************* ** ** Function         NFA_SetNfccMode ** ** Description      This function sets the different NFC controller modes. ** **                  mode ENABLE_MODE_DEFAULT or ENABLE_MODE_TRANSPARENT **                  or ENABLE_MODE_EE ** ** Returns          none ** *******************************************************************************/ extern void NFA_SetNfccMode(uint8_t mode) {  LOG(DEBUG) << StringPrintf("%s: (%d) -> (%d)", __func__, nfc_cb.nfcc_mode,                             mode);  nfc_cb.nfcc_mode = mode; } /******************************************************************************* ** ** Function         NFA_Partial_Init ** ** Description      This function initializes control blocks for NFA based on **                  mode ** **                  p_hal_entry_tbl points to a table of HAL entry points **                  mode ENABLE_MODE_DEFAULT or ENABLE_MODE_TRANSPARENT **                  or ENABLE_MODE_EE ** **                  NOTE: the buffer that p_hal_entry_tbl points must be **                  persistent until NFA is disabled. ** ** Returns          none ** *******************************************************************************/ extern void NFA_Partial_Init(tHAL_NFC_ENTRY* p_hal_entry_tbl, uint8_t mode) {  LOG(DEBUG) << StringPrintf("%s: enter ", __func__);  if (mode == ENABLE_MODE_TRANSPARENT) {    nfa_sys_init();    nfa_dm_init();  } else if (mode == ENABLE_MODE_EE) {    nfa_sys_init();    nfa_dm_init();    nfa_ee_init();  } else {    LOG(ERROR) << StringPrintf("%s: Unknown Mode!", __func__);    return;  }  /* Initialize NFC module */  NFC_Init(p_hal_entry_tbl);  NFA_SetNfccMode(mode);  LOG(DEBUG) << StringPrintf("%s: exit ", __func__); } /******************************************************************************* ** ** Function         NFA_Init ** ** Description      This function initializes control blocks for NFA ** **                  p_hal_entry_tbl points to a table of HAL entry points ** **                  NOTE: the buffer that p_hal_entry_tbl points must be **                  persistent until NFA is disabled. ** ** Returns          none ** *******************************************************************************/ void NFA_Init(tHAL_NFC_ENTRY* p_hal_entry_tbl) {  LOG(VERBOSE) << __func__;  nfa_sys_init();  nfa_dm_init();  nfa_rw_init();  nfa_ce_init();  nfa_ee_init();  if (nfa_ee_max_ee_cfg != 0) {    nfa_dm_cb.get_max_ee = p_hal_entry_tbl->get_max_ee;    nfa_t4tnfcee_init();    nfa_hci_init();  }  nfa_wlc_init();   /* Initialize NFC module */  NFC_Init(p_hal_entry_tbl); } /******************************************************************************* ** ** Function         NFA_Enable ** ** Description      This function enables NFC. Prior to calling NFA_Enable, **                  the NFCC must be powered up, and ready to receive commands. **                  This function enables the tasks needed by NFC, opens the NCI **                  transport, resets the NFC controller, downloads patches to **                  the NFCC (if necessary), and initializes the NFC subsystems. ** **                  This function should only be called once - typically when **                  NFC is enabled during boot-up, or when NFC is enabled from a **                  settings UI. Subsequent calls to NFA_Enable while NFA is **                  enabling or enabled will be ignored. When the NFC startup **                  procedure is completed, an NFA_DM_ENABLE_EVT is returned to **                  the application using the tNFA_DM_CBACK. ** ** Returns          NFA_STATUS_OK if successfully initiated **                  NFA_STATUS_FAILED otherwise ** *******************************************************************************/ tNFA_STATUS NFA_Enable(tNFA_DM_CBACK* p_dm_cback,                       tNFA_CONN_CBACK* p_conn_cback) {  tNFA_DM_API_ENABLE* p_msg;   LOG(VERBOSE) << __func__;   /* Validate parameters */  if ((!p_dm_cback) || (!p_conn_cback)) {    LOG(ERROR) << StringPrintf("%s: error null callback", __func__);    return (NFA_STATUS_FAILED);  }   p_msg = (tNFA_DM_API_ENABLE*)GKI_getbuf(sizeof(tNFA_DM_API_ENABLE));  if (p_msg != nullptr) {    p_msg->hdr.event = NFA_DM_API_ENABLE_EVT;    p_msg->p_dm_cback = p_dm_cback;    p_msg->p_conn_cback = p_conn_cback;     nfa_sys_sendmsg(p_msg);     return (NFA_STATUS_OK);  }   return (NFA_STATUS_FAILED); } /******************************************************************************* ** ** Function         NFA_Disable ** ** Description      This function is called to shutdown NFC. The tasks for NFC **                  are terminated, and clean up routines are performed. This **                  function is typically called during platform shut-down, or **                  when NFC is disabled from a settings UI. When the NFC **                  shutdown procedure is completed, an NFA_DM_DISABLE_EVT is **                  returned to the application using the tNFA_DM_CBACK. ** **                  The platform should wait until the NFC_DISABLE_REVT is **                  received before powering down the NFC chip and NCI **                  transport. This is required to so that NFA can gracefully **                  shut down any open connections. ** ** Returns          NFA_STATUS_OK if successfully initiated **                  NFA_STATUS_FAILED otherwise ** *******************************************************************************/ tNFA_STATUS NFA_Disable(bool graceful) {  tNFA_DM_API_DISABLE* p_msg;   LOG(VERBOSE) << StringPrintf("%s: (graceful=%i)", __func__, graceful);   p_msg = (tNFA_DM_API_DISABLE*)GKI_getbuf(sizeof(tNFA_DM_API_DISABLE));  if (p_msg != nullptr) {    p_msg->hdr.event = NFA_DM_API_DISABLE_EVT;    p_msg->graceful = graceful;     nfa_sys_sendmsg(p_msg);     return (NFA_STATUS_OK);  }   return (NFA_STATUS_FAILED); } /******************************************************************************* ** ** Function         NFA_GetNCIVersion ** ** Description      Returns the NCI version of the NFCC to upper layer ** ** ** Returns          NCI version NCI2.0 / NCI1.0 ** *******************************************************************************/ uint8_t NFA_GetNCIVersion() { return NFC_GetNCIVersion(); } /******************************************************************************* ** ** Function         NFA_SetPowerSubStateForScreenState ** ** Description      Update the power sub-state as per current screen state to **                  NFCC. ** ** Returns          NFA_STATUS_OK if successfully initiated **                  NFA_STATUS_FAILED otherwise ** *******************************************************************************/ tNFA_STATUS NFA_SetPowerSubStateForScreenState(uint8_t screenState) {  LOG(VERBOSE) << StringPrintf("%s: state=0x%X", __func__, screenState);   uint8_t nci_scren_state = 0xFF;  uint16_t buf_size = sizeof(tNFA_DM_API_SET_POWER_SUB_STATE);  tNFA_DM_API_SET_POWER_SUB_STATE* p_msg =      (tNFA_DM_API_SET_POWER_SUB_STATE*)GKI_getbuf(buf_size);   if (p_msg != nullptr) {    p_msg->hdr.event = NFA_DM_API_SET_POWER_SUB_STATE_EVT;    switch (screenState) {      case NFA_SCREEN_STATE_ON_UNLOCKED:        nci_scren_state = SCREEN_STATE_ON_UNLOCKED;        break;      case NFA_SCREEN_STATE_OFF_UNLOCKED:        nci_scren_state = SCREEN_STATE_OFF_UNLOCKED;        break;      case NFA_SCREEN_STATE_ON_LOCKED:        nci_scren_state = SCREEN_STATE_ON_LOCKED;        break;      case NFA_SCREEN_STATE_OFF_LOCKED:        nci_scren_state = SCREEN_STATE_OFF_LOCKED;        break;       default:        LOG(VERBOSE) << StringPrintf("%s: unknown screen state", __func__);        break;    }     p_msg->screen_state = nci_scren_state;     nfa_sys_sendmsg(p_msg);    return (NFA_STATUS_OK);  }  return (NFA_STATUS_FAILED); } /******************************************************************************* ** ** Function         NFA_SetConfig ** ** Description      Set the configuration parameters to NFCC. The result is **                  reported with an NFA_DM_SET_CONFIG_EVT in the tNFA_DM_CBACK **                  callback. ** ** Note:            If RF discovery is started, **                  NFA_StopRfDiscovery()/NFA_RF_DISCOVERY_STOPPED_EVT should **                  happen before calling this function. Most Configuration **                  parameters are related to RF discovery. ** ** Returns          NFA_STATUS_OK if successfully initiated **                  NFA_STATUS_BUSY if previous setting is on-going **                  NFA_STATUS_FAILED otherwise ** *******************************************************************************/ tNFA_STATUS NFA_SetConfig(tNFA_PMID param_id, uint8_t length, uint8_t* p_data) {  tNFA_DM_API_SET_CONFIG* p_msg;   LOG(VERBOSE) << StringPrintf("%s: param_id=0x%X", __func__, param_id);   p_msg = (tNFA_DM_API_SET_CONFIG*)GKI_getbuf(      (uint16_t)(sizeof(tNFA_DM_API_SET_CONFIG) + length));  if (p_msg != nullptr) {    p_msg->hdr.event = NFA_DM_API_SET_CONFIG_EVT;     p_msg->param_id = param_id;    p_msg->length = length;    p_msg->p_data = (uint8_t*)(p_msg + 1);     /* Copy parameter data */    memcpy(p_msg->p_data, p_data, length);     nfa_sys_sendmsg(p_msg);     return (NFA_STATUS_OK);  }   return (NFA_STATUS_FAILED); } /******************************************************************************* ** ** Function         NFA_GetConfig ** ** Description      Get the configuration parameters from NFCC. The result is **                  reported with an NFA_DM_GET_CONFIG_EVT in the tNFA_DM_CBACK **                  callback. ** ** Returns          NFA_STATUS_OK if successfully initiated **                  NFA_STATUS_FAILED otherwise ** *******************************************************************************/ tNFA_STATUS NFA_GetConfig(uint8_t num_ids, tNFA_PMID* p_param_ids) {  tNFA_DM_API_GET_CONFIG* p_msg;   LOG(VERBOSE) << StringPrintf("%s: num_ids=%i", __func__, num_ids);   p_msg = (tNFA_DM_API_GET_CONFIG*)GKI_getbuf(      (uint16_t)(sizeof(tNFA_DM_API_GET_CONFIG) + num_ids));  if (p_msg != nullptr) {    p_msg->hdr.event = NFA_DM_API_GET_CONFIG_EVT;     p_msg->num_ids = num_ids;    p_msg->p_pmids = (tNFA_PMID*)(p_msg + 1);     /* Copy the param IDs */    memcpy(p_msg->p_pmids, p_param_ids, num_ids);     nfa_sys_sendmsg(p_msg);     return (NFA_STATUS_OK);  }   return (NFA_STATUS_FAILED); } /******************************************************************************* ** ** Function         NFA_RequestExclusiveRfControl ** ** Description      Request exclusive control of NFC. **                  - Previous behavior (polling/tag reading, DH card emulation) **                    will be suspended . **                  - Polling and listening will be done based on the specified **                    params ** **                  The NFA_EXCLUSIVE_RF_CONTROL_STARTED_EVT event of **                  tNFA_CONN_CBACK indicates the status of the operation. ** **                  NFA_ACTIVATED_EVT and NFA_DEACTIVATED_EVT indicates link **                  activation/deactivation. ** **                  NFA_SendRawFrame is used to send data to the peer. **                  NFA_DATA_EVT indicates data from the peer. ** **                  If a tag is activated, then the NFA_RW APIs may be used to **                  send commands to the tag. Incoming NDEF messages are sent to **                  the NDEF callback. ** ** Note:            If RF discovery is started, **                  NFA_StopRfDiscovery()/NFA_RF_DISCOVERY_STOPPED_EVT should **                  happen before calling this function ** ** Returns          NFA_STATUS_OK if successfully initiated **                  NFA_STATUS_FAILED otherwise ** *******************************************************************************/ tNFA_STATUS NFA_RequestExclusiveRfControl(tNFA_TECHNOLOGY_MASK poll_mask,                                          tNFA_LISTEN_CFG* p_listen_cfg,                                          tNFA_CONN_CBACK* p_conn_cback,                                          tNFA_NDEF_CBACK* p_ndef_cback) {  tNFA_DM_API_REQ_EXCL_RF_CTRL* p_msg;   LOG(VERBOSE) << StringPrintf("%s: poll_mask=0x%x", __func__, poll_mask);   if (!p_conn_cback) {    LOG(ERROR) << StringPrintf("%s: error null callback", __func__);    return (NFA_STATUS_FAILED);  }   p_msg = (tNFA_DM_API_REQ_EXCL_RF_CTRL*)GKI_getbuf(      sizeof(tNFA_DM_API_REQ_EXCL_RF_CTRL));  if (p_msg != nullptr) {    p_msg->hdr.event = NFA_DM_API_REQUEST_EXCL_RF_CTRL_EVT;    p_msg->poll_mask = poll_mask;    p_msg->p_conn_cback = p_conn_cback;    p_msg->p_ndef_cback = p_ndef_cback;     if (p_listen_cfg)      memcpy(&p_msg->listen_cfg, p_listen_cfg, sizeof(tNFA_LISTEN_CFG));    else      memset(&p_msg->listen_cfg, 0x00, sizeof(tNFA_LISTEN_CFG));     nfa_sys_sendmsg(p_msg);     return (NFA_STATUS_OK);  }   return (NFA_STATUS_FAILED); } /******************************************************************************* ** ** Function         NFA_ReleaseExclusiveRfControl ** ** Description      Release exclusive control of NFC. Once released, behavior **                  prior to obtaining exclusive RF control will resume. ** ** Returns          NFA_STATUS_OK if successfully initiated **                  NFA_STATUS_FAILED otherwise ** *******************************************************************************/ tNFA_STATUS NFA_ReleaseExclusiveRfControl(void) {  NFC_HDR* p_msg;   LOG(VERBOSE) << __func__;   if (!nfa_dm_cb.p_excl_conn_cback) {    LOG(ERROR) << StringPrintf(        "%s: Exclusive rf control is not in "        "progress",        __func__);    return (NFA_STATUS_FAILED);  }   p_msg = (NFC_HDR*)GKI_getbuf(sizeof(NFC_HDR));  if (p_msg != nullptr) {    p_msg->event = NFA_DM_API_RELEASE_EXCL_RF_CTRL_EVT;    nfa_sys_sendmsg(p_msg);    return (NFA_STATUS_OK);  }   return (NFA_STATUS_FAILED); } /******************************************************************************* ** ** Function         NFA_EnablePolling ** ** Description      Enable polling for technologies specified by poll_mask. ** **                  The following events (notified using the connection **                  callback registered with NFA_Enable) are generated during **                  polling: ** **                  - NFA_POLL_ENABLED_EVT indicates whether or not polling **                    successfully enabled. **                  - NFA_DISC_RESULT_EVT indicates there are more than one **                    devices, so application must select one of tags by calling **                    NFA_Select(). **                  - NFA_SELECT_RESULT_EVT indicates whether previous selection **                    was successful or not. If it was failed then application **                    must select again or deactivate by calling **                    NFA_Deactivate(). **                  - NFA_ACTIVATED_EVT is generated when an NFC link is **                    activated. **                  - NFA_NDEF_DETECT_EVT is generated if tag is activated **                  - NFA_DEACTIVATED_EVT will be returned after deactivating **                    NFC link. ** ** Note:            If RF discovery is started, **                  NFA_StopRfDiscovery()/NFA_RF_DISCOVERY_STOPPED_EVT should **                  happen before calling this function ** ** Returns          NFA_STATUS_OK if successfully initiated **                  NFA_STATUS_FAILED otherwise ** *******************************************************************************/ tNFA_STATUS NFA_EnablePolling(tNFA_TECHNOLOGY_MASK poll_mask) {  tNFA_DM_API_ENABLE_POLL* p_msg;   LOG(VERBOSE) << StringPrintf("%s: poll_mask=0x%X", __func__, poll_mask);   p_msg = (tNFA_DM_API_ENABLE_POLL*)GKI_getbuf(sizeof(tNFA_DM_API_ENABLE_POLL));  if (p_msg != nullptr) {    p_msg->hdr.event = NFA_DM_API_ENABLE_POLLING_EVT;    p_msg->poll_mask = poll_mask;     nfa_sys_sendmsg(p_msg);     return (NFA_STATUS_OK);  }   return (NFA_STATUS_FAILED); } /******************************************************************************* ** ** Function         NFA_DisablePolling ** ** Description      Disable polling **                  NFA_POLL_DISABLED_EVT will be returned after stopping **                  polling. ** ** Note:            If RF discovery is started, **                  NFA_StopRfDiscovery()/NFA_RF_DISCOVERY_STOPPED_EVT should **                  happen before calling this function ** ** Returns          NFA_STATUS_OK if successfully initiated **                  NFA_STATUS_FAILED otherwise ** *******************************************************************************/ tNFA_STATUS NFA_DisablePolling(void) {  NFC_HDR* p_msg;   LOG(VERBOSE) << __func__;   p_msg = (NFC_HDR*)GKI_getbuf(sizeof(NFC_HDR));  if (p_msg != nullptr) {    p_msg->event = NFA_DM_API_DISABLE_POLLING_EVT;     nfa_sys_sendmsg(p_msg);     return (NFA_STATUS_OK);  }   return (NFA_STATUS_FAILED); } /******************************************************************************* ** ** Function         NFA_EnableListening ** ** Description      Enable listening. **                  NFA_LISTEN_ENABLED_EVT will be returned after listening is **                  allowed. ** **                  The actual listening technologies are specified by other NFA **                  API functions. Such functions include (but not limited to) **                  NFA_CeConfigureUiccListenTech. **                  If NFA_DisableListening () is called to ignore the listening **                  technologies, NFA_EnableListening () is called to restore **                  the listening technologies set by these functions. ** ** Note:            If RF discovery is started, **                  NFA_StopRfDiscovery()/NFA_RF_DISCOVERY_STOPPED_EVT should **                  happen before calling this function ** ** Returns          NFA_STATUS_OK if successfully initiated **                  NFA_STATUS_FAILED otherwise ** *******************************************************************************/ tNFA_STATUS NFA_EnableListening(void) {  NFC_HDR* p_msg;   LOG(VERBOSE) << __func__;   p_msg = (NFC_HDR*)GKI_getbuf(sizeof(NFC_HDR));  if (p_msg != nullptr) {    p_msg->event = NFA_DM_API_ENABLE_LISTENING_EVT;     nfa_sys_sendmsg(p_msg);     return (NFA_STATUS_OK);  }   return (NFA_STATUS_FAILED); } /******************************************************************************* ** ** Function         NFA_DisableListening ** ** Description      Disable listening **                  NFA_LISTEN_DISABLED_EVT will be returned after stopping **                  listening. This function is called to exclude listen at RF **                  discovery. ** ** Note:            If RF discovery is started, **                  NFA_StopRfDiscovery()/NFA_RF_DISCOVERY_STOPPED_EVT should **                  happen before calling this function ** ** Returns          NFA_STATUS_OK if successfully initiated **                  NFA_STATUS_FAILED otherwise ** *******************************************************************************/ tNFA_STATUS NFA_DisableListening(void) {  NFC_HDR* p_msg;   LOG(VERBOSE) << __func__;   p_msg = (NFC_HDR*)GKI_getbuf(sizeof(NFC_HDR));  if (p_msg != nullptr) {    p_msg->event = NFA_DM_API_DISABLE_LISTENING_EVT;     nfa_sys_sendmsg(p_msg);     return (NFA_STATUS_OK);  }   return (NFA_STATUS_FAILED); } /******************************************************************************* ** ** Function         NFA_StartRfDiscovery ** ** Description      Start RF discovery **                  RF discovery parameters shall be set by other APIs. ** **                  An NFA_RF_DISCOVERY_STARTED_EVT indicates whether starting **                  was successful or not. ** ** Returns          NFA_STATUS_OK if successfully initiated **                  NFA_STATUS_FAILED otherwise ** *******************************************************************************/ tNFA_STATUS NFA_StartRfDiscovery(void) {  NFC_HDR* p_msg;   LOG(VERBOSE) << __func__;   p_msg = (NFC_HDR*)GKI_getbuf(sizeof(NFC_HDR));  if (p_msg != nullptr) {    p_msg->event = NFA_DM_API_START_RF_DISCOVERY_EVT;     nfa_sys_sendmsg(p_msg);     return (NFA_STATUS_OK);  }   return (NFA_STATUS_FAILED); } /******************************************************************************* ** ** Function         NFA_StopRfDiscovery ** ** Description      Stop RF discovery ** **                  An NFA_RF_DISCOVERY_STOPPED_EVT indicates whether stopping **                  was successful or not. ** ** Returns          NFA_STATUS_OK if successfully initiated **                  NFA_STATUS_FAILED otherwise ** *******************************************************************************/ tNFA_STATUS NFA_StopRfDiscovery(void) {  NFC_HDR* p_msg;   LOG(VERBOSE) << __func__;   p_msg = (NFC_HDR*)GKI_getbuf(sizeof(NFC_HDR));  if (p_msg != nullptr) {    p_msg->event = NFA_DM_API_STOP_RF_DISCOVERY_EVT;     nfa_sys_sendmsg(p_msg);     return (NFA_STATUS_OK);  }   return (NFA_STATUS_FAILED); } /******************************************************************************* ** ** Function         NFA_SetRfDiscoveryDuration ** ** Description      Set the duration of the single discovery period in [ms]. **                  Allowable range: 0 ms to 0xFFFF ms. ** **                  If discovery is already started, the application should **                  call NFA_StopRfDiscovery prior to calling **                  NFA_SetRfDiscoveryDuration, and then call **                  NFA_StartRfDiscovery afterwards to restart discovery using **                  the new duration. ** ** Note:            If RF discovery is started, **                  NFA_StopRfDiscovery()/NFA_RF_DISCOVERY_STOPPED_EVT should **                  happen before calling this function ** ** Returns: **                  NFA_STATUS_OK, if command accepted **                  NFA_STATUS_FAILED: otherwise ** *******************************************************************************/ tNFA_STATUS NFA_SetRfDiscoveryDuration(uint16_t discovery_period_ms) {  tNFA_DM_API_SET_RF_DISC_DUR* p_msg;   LOG(VERBOSE) << __func__;   /* Post the API message */  p_msg = (tNFA_DM_API_SET_RF_DISC_DUR*)GKI_getbuf(      sizeof(tNFA_DM_API_SET_RF_DISC_DUR));  if (p_msg != nullptr) {    p_msg->hdr.event = NFA_DM_API_SET_RF_DISC_DURATION_EVT;     /* Set discovery duration */    p_msg->rf_disc_dur_ms = discovery_period_ms;     nfa_sys_sendmsg(p_msg);     return (NFA_STATUS_OK);  }   return (NFA_STATUS_FAILED); } /******************************************************************************* ** ** Function         NFA_Select ** ** Description      Select one from detected devices during discovery **                  (from NFA_DISC_RESULT_EVTs). The application should wait for **                  the final NFA_DISC_RESULT_EVT before selecting. ** **                  An NFA_SELECT_RESULT_EVT indicates whether selection was **                  successful or not. If failed then application must select **                  again or deactivate by NFA_Deactivate(). ** ** Returns          NFA_STATUS_OK if successfully initiated **                  NFA_STATUS_INVALID_PARAM if RF interface is not matched **                  protocol **                  NFA_STATUS_FAILED otherwise ** *******************************************************************************/ tNFA_STATUS NFA_Select(uint8_t rf_disc_id, tNFA_NFC_PROTOCOL protocol,                       tNFA_INTF_TYPE rf_interface) {  tNFA_DM_API_SELECT* p_msg;   LOG(VERBOSE) << StringPrintf(      "%s: rf_disc_id=0x%X, protocol=0x%X, rf_interface=0x%X", __func__,      rf_disc_id, protocol, rf_interface);   if ((rf_interface == NFA_INTERFACE_ISO_DEP) &&      (protocol != NFA_PROTOCOL_ISO_DEP)) {    LOG(ERROR) << StringPrintf("%s:  RF interface is not matched protocol",                               __func__);    return (NFA_STATUS_INVALID_PARAM);  }   p_msg =      (tNFA_DM_API_SELECT*)GKI_getbuf((uint16_t)(sizeof(tNFA_DM_API_SELECT)));  if (p_msg != nullptr) {    p_msg->hdr.event = NFA_DM_API_SELECT_EVT;    p_msg->rf_disc_id = rf_disc_id;    p_msg->protocol = protocol;    p_msg->rf_interface = rf_interface;     nfa_sys_sendmsg(p_msg);     return (NFA_STATUS_OK);  }   return (NFA_STATUS_FAILED); } /******************************************************************************* ** ** Function         NFA_UpdateRFCommParams ** ** Description      This function is called to update RF Communication **                  parameters once the Frame RF Interface has been activated. ** **                  An NFA_UPDATE_RF_PARAM_RESULT_EVT indicates whether updating **                  was successful or not. ** ** Returns          NFA_STATUS_OK if successfully initiated **                  NFA_STATUS_FAILED otherwise ** *******************************************************************************/ tNFA_STATUS NFA_UpdateRFCommParams(tNFA_RF_COMM_PARAMS* p_params) {  tNFA_DM_API_UPDATE_RF_PARAMS* p_msg;   LOG(VERBOSE) << __func__;   p_msg = (tNFA_DM_API_UPDATE_RF_PARAMS*)GKI_getbuf(      (uint16_t)(sizeof(tNFA_DM_API_UPDATE_RF_PARAMS)));  if (p_msg != nullptr) {    p_msg->hdr.event = NFA_DM_API_UPDATE_RF_PARAMS_EVT;    memcpy(&p_msg->params, p_params, sizeof(tNFA_RF_COMM_PARAMS));     nfa_sys_sendmsg(p_msg);     return (NFA_STATUS_OK);  }   return (NFA_STATUS_FAILED); } /******************************************************************************* ** ** Function         NFA_StartRemovalDetection ** ** Description      Start the detection of RF endpoint removal when data *exchange **                  ore charging with listener completed. ** **                  An NFA_DETECT_REMOVAL_RESULT_EVT indicates whether start **                  was successful or not. ** ** Returns          NFA_STATUS_OK if successfully initiated **                  NFA_STATUS_FAILED otherwise ** *******************************************************************************/ tNFA_STATUS NFA_StartRemovalDetection(uint8_t waiting_time_int) {  tNFA_DM_API_START_REMOVAL_DETECT* p_msg;   LOG(VERBOSE) << StringPrintf("%s:  waiting_time_int=0x%X", __func__,                               waiting_time_int);   p_msg = (tNFA_DM_API_START_REMOVAL_DETECT*)GKI_getbuf(      (uint8_t)(sizeof(tNFA_DM_API_START_REMOVAL_DETECT)));  if (p_msg != nullptr) {    p_msg->hdr.event = NFA_DM_API_START_REMOVAL_DETECT_EVT;    p_msg->waiting_time_int = waiting_time_int;    nfa_sys_sendmsg(p_msg);     return (NFA_STATUS_OK);  }   return (NFA_STATUS_FAILED); } /******************************************************************************* ** ** Function         NFA_Deactivate ** ** Description **                  If sleep_mode=TRUE: **                      Deselect the activated device by deactivating into sleep **                      mode. ** **                      An NFA_DEACTIVATE_FAIL_EVT indicates that selection was **                      not successful. Application can select another **                      discovered device or deactivate by NFA_Deactivate() **                      after receiving NFA_DEACTIVATED_EVT. ** **                      Deactivating to sleep mode is not allowed when NFCC is **                      in wait-for-host-select mode, or in listen-sleep states; **                      NFA will deactivate to idle or discovery state for these **                      cases respectively. ** ** **                  If sleep_mode=FALSE: **                      Deactivate the connection (e.g. as a result of presence **                      check failure) NFA_DEACTIVATED_EVT will indicate that **                      link is deactivated. Polling/listening will resume **                      (unless the nfcc is in wait_for-all-discoveries state) ** ** ** Returns          NFA_STATUS_OK if successfully initiated **                  NFA_STATUS_FAILED otherwise ** *******************************************************************************/ extern tNFA_STATUS NFA_Deactivate(bool sleep_mode) {  tNFA_DM_API_DEACTIVATE* p_msg;   LOG(VERBOSE) << StringPrintf("%s: sleep_mode=%i", __func__, sleep_mode);   p_msg = (tNFA_DM_API_DEACTIVATE*)GKI_getbuf(      (uint16_t)(sizeof(tNFA_DM_API_DEACTIVATE)));  if (p_msg != nullptr) {    p_msg->hdr.event = NFA_DM_API_DEACTIVATE_EVT;    p_msg->sleep_mode = sleep_mode;     nfa_sys_sendmsg(p_msg);     return (NFA_STATUS_OK);  }   return (NFA_STATUS_FAILED); } /******************************************************************************* ** ** Function         NFA_SendRawFrame ** ** Description      Send a raw frame over the activated interface with the NFCC. **                  This function can only be called after NFC link is **                  activated. ** **                  If the activated interface is a tag and auto-presence check **                  is enabled then presence_check_start_delay can be used to **                  indicate the delay in msec after which the next auto **                  presence check command can be sent. **                  NFA_DM_DEFAULT_PRESENCE_CHECK_START_DELAY can be used as the **                  default value for the delay. ** ** Returns          NFA_STATUS_OK if successfully initiated **                  NFA_STATUS_FAILED otherwise ** *******************************************************************************/ tNFA_STATUS NFA_SendRawFrame(uint8_t* p_raw_data, uint16_t data_len,                             uint16_t presence_check_start_delay) {  NFC_HDR* p_msg;  uint16_t size;  uint8_t* p;   LOG(VERBOSE) << StringPrintf("%s: data_len=%d", __func__, data_len);   size = NFC_HDR_SIZE + NCI_MSG_OFFSET_SIZE + NCI_DATA_HDR_SIZE + data_len;  /* Check for integer overflow */  if (size < data_len) {    android_errorWriteLog(0x534e4554, "120664978");    return NFA_STATUS_INVALID_PARAM;  }  p_msg = (NFC_HDR*)GKI_getbuf(size);  if (p_msg != nullptr) {    p_msg->event = NFA_DM_API_RAW_FRAME_EVT;    p_msg->layer_specific = presence_check_start_delay;    p_msg->offset = NCI_MSG_OFFSET_SIZE + NCI_DATA_HDR_SIZE;    p_msg->len = data_len;     p = (uint8_t*)(p_msg + 1) + p_msg->offset;    if ((data_len != 0) && (p_raw_data != nullptr)) {      memcpy(p, p_raw_data, data_len);    }     nfa_sys_sendmsg(p_msg);     return (NFA_STATUS_OK);  }   return (NFA_STATUS_FAILED); } /******************************************************************************* ** NDEF Handler APIs *******************************************************************************/ /******************************************************************************* ** ** Function         NFA_RegisterNDefTypeHandler ** ** Description      This function allows the applications to register for **                  specific types of NDEF records. When NDEF records are **                  received, NFA will parse the record-type field, and pass **                  the record to the registered tNFA_NDEF_CBACK. ** **                  For records types which were not registered, the record will **                  be sent to the default handler. A default type-handler may **                  be registered by calling this NFA_RegisterNDefTypeHandler **                  with tnf=NFA_TNF_DEFAULT. In this case, all un-registered **                  record types will be sent to the callback. Only one default **                  handler may be registered at a time. ** **                  An NFA_NDEF_REGISTER_EVT will be sent to the tNFA_NDEF_CBACK **                  to indicate that registration was successful, and provide a **                  handle for this record type. ** ** Returns          NFA_STATUS_OK if successfully initiated **                  NFA_STATUS_FAILED otherwise ** *******************************************************************************/ tNFA_STATUS NFA_RegisterNDefTypeHandler(bool handle_whole_message, tNFA_TNF tnf,                                        uint8_t* p_type_name,                                        uint8_t type_name_len,                                        tNFA_NDEF_CBACK* p_ndef_cback) {  tNFA_DM_API_REG_NDEF_HDLR* p_msg;   LOG(VERBOSE) << StringPrintf(      "%s: handle whole ndef message=%i, "      "tnf=0x%02x",      __func__, handle_whole_message, tnf);   /* Check for NULL callback */  if (!p_ndef_cback) {    LOG(ERROR) << StringPrintf("%s: error - null callback", __func__);    return (NFA_STATUS_INVALID_PARAM);  }   p_msg = (tNFA_DM_API_REG_NDEF_HDLR*)GKI_getbuf(      (uint16_t)(sizeof(tNFA_DM_API_REG_NDEF_HDLR) + type_name_len));  if (p_msg != nullptr) {    p_msg->hdr.event = NFA_DM_API_REG_NDEF_HDLR_EVT;     p_msg->flags =        (handle_whole_message ? NFA_NDEF_FLAGS_HANDLE_WHOLE_MESSAGE : 0);    p_msg->tnf = tnf;    p_msg->name_len = type_name_len;    p_msg->p_ndef_cback = p_ndef_cback;    memcpy(p_msg->name, p_type_name, type_name_len);     nfa_sys_sendmsg(p_msg);     return (NFA_STATUS_OK);  }   return (NFA_STATUS_FAILED); } /******************************************************************************* ** ** Function         NFA_RegisterNDefUriHandler ** ** Description      This API is a special-case of NFA_RegisterNDefTypeHandler **                  with TNF=NFA_TNF_WKT, and type_name='U' (URI record); and **                  allows registering for specific URI types (e.g. 'tel:' or **                  'mailto:'). ** **                  An NFA_NDEF_REGISTER_EVT will be sent to the tNFA_NDEF_CBACK **                  to indicate that registration was successful, and provide a **                  handle for this registration. ** **                  If uri_id=NFA_NDEF_URI_ID_ABSOLUTE, then p_abs_uri contains **                  the unabridged URI. For all other uri_id values, **                  the p_abs_uri parameter is ignored (i.e the URI prefix is **                  implied by uri_id). See [NFC RTD URI] for more information. ** ** Returns          NFA_STATUS_OK if successfully initiated **                  NFA_STATUS_FAILED otherwise ** *******************************************************************************/ extern tNFA_STATUS NFA_RegisterNDefUriHandler(bool handle_whole_message,                                              tNFA_NDEF_URI_ID uri_id,                                              uint8_t* p_abs_uri,                                              uint8_t uri_id_len,                                              tNFA_NDEF_CBACK* p_ndef_cback) {  tNFA_DM_API_REG_NDEF_HDLR* p_msg;   LOG(VERBOSE) << StringPrintf(      "%s: handle whole ndef message=%i, "      "uri_id=0x%02x",      __func__, handle_whole_message, uri_id);   /* Check for NULL callback */  if (!p_ndef_cback) {    LOG(ERROR) << StringPrintf("%s: error - null callback", __func__);    return (NFA_STATUS_INVALID_PARAM);  }   p_msg = (tNFA_DM_API_REG_NDEF_HDLR*)GKI_getbuf(      (uint16_t)(sizeof(tNFA_DM_API_REG_NDEF_HDLR) + uri_id_len));  if (p_msg != nullptr) {    p_msg->hdr.event = NFA_DM_API_REG_NDEF_HDLR_EVT;     p_msg->flags = NFA_NDEF_FLAGS_WKT_URI;     if (handle_whole_message) {      p_msg->flags |= NFA_NDEF_FLAGS_HANDLE_WHOLE_MESSAGE;    }     /* abs_uri is only valid fir uri_id=NFA_NDEF_URI_ID_ABSOLUTE */    if (uri_id != NFA_NDEF_URI_ID_ABSOLUTE) {      uri_id_len = 0;    }     p_msg->tnf = NFA_TNF_WKT;    p_msg->uri_id = uri_id;    p_msg->name_len = uri_id_len;    p_msg->p_ndef_cback = p_ndef_cback;    memcpy(p_msg->name, p_abs_uri, uri_id_len);     nfa_sys_sendmsg(p_msg);     return (NFA_STATUS_OK);  }   return (NFA_STATUS_FAILED); } /******************************************************************************* ** ** Function         NFA_DeregisterNDefTypeHandler ** ** Description      Deregister NDEF record type handler. ** ** Returns          NFA_STATUS_OK if successfully initiated **                  NFA_STATUS_FAILED otherwise ** *******************************************************************************/ extern tNFA_STATUS NFA_DeregisterNDefTypeHandler(tNFA_HANDLE ndef_type_handle) {  tNFA_DM_API_DEREG_NDEF_HDLR* p_msg;   LOG(VERBOSE) << StringPrintf("%s: handle 0x%08x", __func__, ndef_type_handle);   p_msg = (tNFA_DM_API_DEREG_NDEF_HDLR*)GKI_getbuf(      (uint16_t)(sizeof(tNFA_DM_API_DEREG_NDEF_HDLR)));  if (p_msg != nullptr) {    p_msg->hdr.event = NFA_DM_API_DEREG_NDEF_HDLR_EVT;    p_msg->ndef_type_handle = ndef_type_handle;     nfa_sys_sendmsg(p_msg);     return (NFA_STATUS_OK);  }   return (NFA_STATUS_FAILED); } /******************************************************************************* ** ** Function         NFA_PowerOffSleepMode ** ** Description      This function is called to enter or leave NFCC Power Off **                  Sleep mode NFA_DM_PWR_MODE_CHANGE_EVT will be sent to **                  indicate status. ** **                  start_stop : TRUE if entering Power Off Sleep mode **                               FALSE if leaving Power Off Sleep mode ** ** Returns          NFA_STATUS_OK if successfully initiated **                  NFA_STATUS_FAILED otherwise ** *******************************************************************************/ tNFA_STATUS NFA_PowerOffSleepMode(bool start_stop) {  NFC_HDR* p_msg;   LOG(VERBOSE) << StringPrintf("%s: start_stop=%d", __func__, start_stop);   if (nfa_dm_cb.flags & NFA_DM_FLAGS_SETTING_PWR_MODE) {    LOG(ERROR) << StringPrintf("%s: NFA DM is busy to update power mode",                               __func__);    return (NFA_STATUS_FAILED);  } else {    nfa_dm_cb.flags |= NFA_DM_FLAGS_SETTING_PWR_MODE;  }   p_msg = (NFC_HDR*)GKI_getbuf(sizeof(NFC_HDR));  if (p_msg != nullptr) {    p_msg->event = NFA_DM_API_POWER_OFF_SLEEP_EVT;    p_msg->layer_specific = start_stop;     nfa_sys_sendmsg(p_msg);     return (NFA_STATUS_OK);  }   return (NFA_STATUS_FAILED); } /******************************************************************************* ** ** Function         NFA_RegVSCback ** ** Description      This function is called to register or de-register a **                  callback function to receive Proprietary NCI response and **                  notification events. The maximum number of callback **                  functions allowed is NFC_NUM_VS_CBACKS ** ** Returns          tNFC_STATUS ** *******************************************************************************/ tNFC_STATUS NFA_RegVSCback(bool is_register, tNFA_VSC_CBACK* p_cback) {  tNFA_DM_API_REG_VSC* p_msg;   LOG(VERBOSE) << StringPrintf("%s: is_register=%d", __func__, is_register);   if (p_cback == nullptr) {    LOG(ERROR) << StringPrintf("%s: requires a valid callback function",                               __func__);    return (NFA_STATUS_FAILED);  }   p_msg = (tNFA_DM_API_REG_VSC*)GKI_getbuf(sizeof(tNFA_DM_API_REG_VSC));  if (p_msg != nullptr) {    p_msg->hdr.event = NFA_DM_API_REG_VSC_EVT;    p_msg->is_register = is_register;    p_msg->p_cback = p_cback;     nfa_sys_sendmsg(p_msg);     return (NFA_STATUS_OK);  }   return (NFA_STATUS_FAILED); } /******************************************************************************* ** ** Function         NFA_SendVsCommand ** ** Description      This function is called to send an NCI Vendor Specific **                  command to NFCC. ** **                  oid             - The opcode of the VS command. **                  cmd_params_len  - The command parameter len **                  p_cmd_params    - The command parameter **                  p_cback         - The callback function to receive the **                                    command status ** ** Returns          NFA_STATUS_OK if successfully initiated **                  NFA_STATUS_FAILED otherwise ** *******************************************************************************/ tNFA_STATUS NFA_SendVsCommand(uint8_t oid, uint8_t cmd_params_len,                              uint8_t* p_cmd_params, tNFA_VSC_CBACK* p_cback) {  tNFA_DM_API_SEND_VSC* p_msg;  uint16_t size = sizeof(tNFA_DM_API_SEND_VSC) + cmd_params_len;   LOG(VERBOSE) << StringPrintf("%s: oid=0x%x", __func__, oid);   p_msg = (tNFA_DM_API_SEND_VSC*)GKI_getbuf(size);  if (p_msg != nullptr) {    p_msg->hdr.event = NFA_DM_API_SEND_VSC_EVT;    p_msg->oid = oid;    p_msg->p_cback = p_cback;    if (cmd_params_len && p_cmd_params) {      p_msg->cmd_params_len = cmd_params_len;      p_msg->p_cmd_params = (uint8_t*)(p_msg + 1);      memcpy(p_msg->p_cmd_params, p_cmd_params, cmd_params_len);    } else {      p_msg->cmd_params_len = 0;      p_msg->p_cmd_params = nullptr;    }     nfa_sys_sendmsg(p_msg);     return (NFA_STATUS_OK);  }   return (NFA_STATUS_FAILED); } /******************************************************************************* ** ** Function         NFA_SendRawVsCommand ** ** Description      This function is called to send raw Vendor Specific **                  command to NFCC. ** **                  cmd_params_len  - The command parameter len **                  p_cmd_params    - The command parameter **                  p_cback         - The callback function to receive the **                                    command ** ** Returns          NFA_STATUS_OK if successfully initiated **                  NFA_STATUS_FAILED otherwise ** *******************************************************************************/ tNFA_STATUS NFA_SendRawVsCommand(uint8_t cmd_params_len, uint8_t* p_cmd_params,                                 tNFA_VSC_CBACK* p_cback) {  if (cmd_params_len == 0x00 || p_cmd_params == nullptr || p_cback == nullptr) {    return NFA_STATUS_INVALID_PARAM;  }  uint16_t size = sizeof(tNFA_DM_API_SEND_VSC) + cmd_params_len;  tNFA_DM_API_SEND_VSC* p_msg = (tNFA_DM_API_SEND_VSC*)GKI_getbuf(size);   if (p_msg != nullptr) {    p_msg->hdr.event = NFA_DM_API_SEND_RAW_VS_EVT;    p_msg->p_cback = p_cback;    p_msg->cmd_params_len = cmd_params_len;    p_msg->p_cmd_params = (uint8_t*)(p_msg + 1);    memcpy(p_msg->p_cmd_params, p_cmd_params, cmd_params_len);     nfa_sys_sendmsg(p_msg);     return NFA_STATUS_OK;  }   return NFA_STATUS_FAILED; } /******************************************************************************* ** ** Function:        NFA_EnableDtamode ** ** Description:     Enable DTA Mode ** ** Returns:         none: ** *******************************************************************************/ void NFA_EnableDtamode(tNFA_eDtaModes eDtaMode) {  LOG(VERBOSE) << StringPrintf("%s: eDtaMode=0x%x ", __func__, eDtaMode);  appl_dta_mode_flag = 0x01;  nfa_dm_cb.eDtaMode = eDtaMode; } /******************************************************************************* ** ** Function:        NFA_DisableDtamode ** ** Description:     Disable DTA Mode ** ** Returns:         none: ** *******************************************************************************/ void NFA_DisableDtamode(void) {  LOG(VERBOSE) << StringPrintf("%s: enter", __func__);  appl_dta_mode_flag = 0x0;  nfa_dm_cb.eDtaMode = NFA_DTA_APPL_MODE; } /******************************************************************************* ** ** Function:        NFA_SetNfcSecure ** ** Description:     Prooagtes NFC secure settings to NFC TASK ** ** Returns:         none: ** *******************************************************************************/ void NFA_SetNfcSecure(bool status) {  LOG(DEBUG) << StringPrintf("%s:  status=%d", __func__, status);  nfa_dm_cb.is_nfc_secure = status; } /******************************************************************************* ** ** Function         NFA_ChangeDiscoveryTech ** ** Description      Change RF discoverying technologies specified by **                  pollTech and listenTech. ** **                  NFA_DM_API_CHANGE_DISCOVERY_TECH_EVT will be returned. ** **                  If pollTech/listenTech are set to 0xFF, it revert to **                  the polling/listening technologies **                  before NFA_ChangeDiscoveryTech() was called. ** **                  is_revert_poll : TRUE if reverting RF polling tech **                               before calling NFA_StopRfDiscovery **                               FALSE if changing RF polling tech according **                               to pollTech **                  is_revert_listen : TRUE if reverting RF listening tech **                               before calling NFA_StopRfDiscovery **                               FALSE if changing RF listening tech according **                               to listenTech **                  change_default_tech : TRUE if the default technolofy mask **                               has to be changed according to listenTech **                               and pollTech settings ** ** Note:            If RF discovery is started, **                  NFA_StopRfDiscovery()/NFA_RF_DISCOVERY_STOPPED_EVT **                  should happen before calling this function ** ** Returns          NFA_STATUS_OK if successfully initiated **                  NFA_STATUS_FAILED otherwise ** *******************************************************************************/ tNFA_STATUS NFA_ChangeDiscoveryTech(tNFA_TECHNOLOGY_MASK pollTech,                                    tNFA_TECHNOLOGY_MASK listenTech,                                    bool is_revert_poll, bool is_revert_listen,                                    bool change_default_tech) {  tNFA_DM_API_CHANGE_DISCOVERY_TECH* p_msg;  LOG(VERBOSE) << StringPrintf("%s: pollTech=0x%X listenTech=0x%X", __func__,                               pollTech, listenTech);   if ((p_msg = (tNFA_DM_API_CHANGE_DISCOVERY_TECH*)GKI_getbuf(           sizeof(tNFA_DM_API_CHANGE_DISCOVERY_TECH))) != nullptr) {    p_msg->hdr.event = NFA_DM_API_CHANGE_DISCOVERY_TECH_EVT;    p_msg->is_revert_poll = is_revert_poll;    p_msg->is_revert_listen = is_revert_listen;    p_msg->change_default_tech = change_default_tech;    p_msg->change_poll_mask = pollTech;    p_msg->change_listen_mask = listenTech;     nfa_sys_sendmsg(p_msg);     return (NFA_STATUS_OK);  }   return (NFA_STATUS_FAILED); } /******************************************************************************* ** ** Function         NFA_IsRfRemovalDetectionSupported ** ** Description      Indicates if RF Removal Detection mode is supported by NFCC **                  and by activated RF Interface. ** ** Returns          true if supported else false. ** *******************************************************************************/ bool NFA_IsRfRemovalDetectionSupported() {  return NFC_IsRfRemovalDetectionSupported(); } nfa的作用和定位，以及nfa_init（）,和nfa_enable（）是怎么工作的









已深度思考（用时 27 秒）



NFA（NFC Forum Adaptation Layer）是NFC协议栈中的一个重要组件，位于NCI（NFC Controller Interface）之上，为上层应用提供统一的API接口。NFA的作用是抽象和管理底层的NFC控制器操作，包括设备发现、连接管理、数据传输等，同时处理多个NFC应用之间的协调。

\### NFA的定位：

\- **在协议栈中的位置**：NFA位于NCI层之上，应用层之下。它接收来自应用层的请求，将其转换为NCI命令发送给NFCC（NFC Controller），同时将NFCC的事件和通知传递给应用层。

\- **功能**：NFA负责初始化NFC子系统、管理RF发现过程、处理设备激活与数据交换、管理NFCEE（NFC Execution Environment）等。

\### `NFA_Init()`函数：

\- **作用**：初始化NFA模块的控制块（control blocks）。这个函数在NFC协议栈启动时被调用，用于设置NFA内部各个子模块（如设备管理、读写模式、卡模拟等）的初始状态。

\- **工作流程**：

1. 调用`nfa_sys_init()`初始化NFA系统。
2. 调用各个子模块的初始化函数，如`nfa_dm_init()`（设备管理）、`nfa_rw_init()`（读写模式）、`nfa_ce_init()`（卡模拟）等。
3. 根据配置（如`nfa_ee_max_ee_cfg`）决定是否初始化其他模块（如T4T NFCEE、HCI等）。
4. 最后调用`NFC_Init()`初始化NFC模块（底层NCI传输和控制器交互）。

\### `NFA_Enable()`函数：

\- **作用**：启用NFC功能。这个函数在`NFA_Init()`之后调用，用于启动NFC控制器并准备接收命令。它会打开NCI传输、重置NFCC、下载固件补丁（如果需要），并初始化NFC子系统。

\- **工作流程**：

1. 检查传入的回调函数指针是否有效（设备管理回调和连接回调）。
2. 分配一个消息缓冲区（`tNFA_DM_API_ENABLE`），设置消息类型为`NFA_DM_API_ENABLE_EVT`，并保存回调函数。
3. 将消息发送到NFA系统任务（通过`nfa_sys_sendmsg`）。
4. 在NFA系统任务中处理该消息，最终会通过NCI命令序列初始化NFCC，完成后通过`NFA_DM_ENABLE_EVT`通知应用层。

\### 总结：

\- **`NFA_Init()`**：初始化NFA内部的数据结构和子模块，为NFC操作做准备。

\- **`NFA_Enable()`**：启动NFC功能，包括与NFCC的通信初始化，完成后通知应用层可以开始使用NFC服务。

这两个函数是NFC协议栈启动的关键步骤，确保硬件和软件都准备好处理后续的NFC操作。

### NFA的作用和定位

NFA（NFC Forum Adaptation Layer）是NFC协议栈的核心层，位于应用层和底层NCI（NFC Controller Interface）之间。其主要作用和定位包括：

1. **协议适配**：
   提供统一的API接口给上层应用，屏蔽底层NCI细节。
2. **状态管理**：
   管理NFC设备状态（发现、激活、休眠等）。
3. **事件分发**：
   处理来自NFCC（NFC Controller）的事件并分发给应用。
4. **多应用协调**：
   协调多个NFC应用对硬件资源的访问。
5. **安全控制**：
   处理安全相关操作（如Secure Element访问）。

------

### `NFA_Init()`的工作流程

c



复制



下载

```
void NFA_Init(tHAL_NFC_ENTRY* p_hal_entry_tbl) {
  nfa_sys_init();          // 初始化NFA任务调度系统
  nfa_dm_init();           // 初始化设备管理模块
  nfa_rw_init();           // 初始化读写器模块
  nfa_ce_init();           // 初始化卡模拟模块
  nfa_ee_init();           // 初始化执行环境模块
  if (nfa_ee_max_ee_cfg != 0) {
    nfa_dm_cb.get_max_ee = p_hal_entry_tbl->get_max_ee;
    nfa_t4tnfcee_init();   // 初始化Type 4标签
    nfa_hci_init();        // 初始化主机控制器接口
  }
  nfa_wlc_init();          // 初始化无线充电模块
  NFC_Init(p_hal_entry_tbl); // 初始化底层NFC控制器
}
```

**关键步骤**：

1. **子系统初始化**：
   初始化NFA内部各子模块（设备管理、读写器、卡模拟等）。
2. **硬件抽象层绑定**：
   通过`p_hal_entry_tbl`注册HAL层函数指针。
3. **条件初始化**：
   根据配置动态初始化T4T/HCI等可选模块。
4. **NFCC初始化**：
   调用`NFC_Init()`初始化底层NFC控制器。

------

### `NFA_Enable()`的工作流程

c



复制



下载

```
tNFA_STATUS NFA_Enable(tNFA_DM_CBACK* p_dm_cback,
                       tNFA_CONN_CBACK* p_conn_cback) {
  // 验证回调函数
  if ((!p_dm_cback) || (!p_conn_cback)) 
    return NFA_STATUS_FAILED;

  // 创建消息
  tNFA_DM_API_ENABLE* p_msg = ...;
  p_msg->hdr.event = NFA_DM_API_ENABLE_EVT;
  p_msg->p_dm_cback = p_dm_cback;      // 设备管理回调
  p_msg->p_conn_cback = p_conn_cback;  // 连接事件回调

  // 发送到NFA任务队列
  nfa_sys_sendmsg(p_msg);
  return NFA_STATUS_OK;
}
```

**执行流程**：

1. **参数校验**：
   确保设备管理回调和连接回调有效。

2. **异步消息构造**：
   创建`NFA_DM_API_ENABLE_EVT`事件消息。

3. **回调注册**：
   保存应用层提供的两个关键回调：

   - `p_dm_cback`：处理设备管理事件（启停状态）。
   - `p_conn_cback`：处理连接事件（激活/数据交换）。

4. **事件派发**：
   消息发送到NFA系统任务队列异步执行。

5. **底层激活**：
   在NFA任务中实际执行：

   c

   

   复制

   

   下载

   ```
   // nfa_dm_act.c
   case NFA_DM_API_ENABLE_EVT:
     nfa_dm_enable(p_msg); // 启动NFCC、下载固件等
     break;
   ```

6. **结果通知**：
   完成后通过`p_dm_cback`发送`NFA_DM_ENABLE_EVT`通知应用。

------

### 关键设计特点

1. **异步模型**：
   所有API通过消息队列异步执行，避免阻塞调用线程。
2. **回调机制**：
   通过分层回调（Device Management + Connection）解耦事件处理。
3. **模块化**：
   子模块（CE/EE/RW）独立初始化，通过任务调度协同工作。
4. **条件编译**：
   根据`nfa_ee_max_ee_cfg`动态启用可选功能（如HCI）。

> **定位总结**：NFA作为协议栈中枢，承上（应用）启下（NFCC），通过消息驱动和回调机制实现高效异步控制，核心价值在于提供稳定、可扩展的NFC服务抽象层。









NFC_Enable(nfa_dm_nfc_response_cback)的作用









已深度思考（用时 17 秒）



我们之前已经讨论了NFA_Init和NFA_Enable的工作流程。现在问题聚焦在NFC_Enable函数的作用，并且传入的参数是nfa_dm_nfc_response_cback。

 根据上下文，NFC_Enable是底层NFC模块的一个函数，它由NFA层调用，用于启用NFC控制器（NFCC）并设置回调函数。

 在NFA_Enable函数中，我们创建了一个消息（NFA_DM_API_ENABLE_EVT）发送到NFA系统任务。在任务处理中，会调用nfa_dm_enable函数。

 在nfa_dm_enable函数中，会进一步调用底层的NFC_Enable函数，并传入一个回调函数（nfa_dm_nfc_response_cback）用于接收来自底层的事件。

 因此，NFC_Enable的作用是：

1. 初始化NFCC，包括硬件复位、固件下载（如果需要）等。
2. 设置一个回调函数（nfa_dm_nfc_response_cback），该回调函数用于将底层NFC模块（NFC）产生的事件传递给NFA层。

 具体来说，当底层NFC模块完成初始化、收到数据、状态变化等事件时，会通过这个回调函数通知NFA层。

 在代码中，NFC_Enable的函数原型可能是这样的（在nfc_int.h中可能有定义）：

   void NFC_Enable(tNFC_RESPONSE_CBACK *p_cback);

 其中，tNFC_RESPONSE_CBACK是底层事件回调函数的类型定义。

 在NFA层，nfa_dm_nfc_response_cback函数负责处理这些底层事件，并将其转化为NFA层的事件，然后通过之前注册的nfa_dm_cb.p_dm_cback和nfa_dm_cb.p_conn_cback传递给应用层。

 所以，NFC_Enable(nfa_dm_nfc_response_cback)的作用是启用底层NFC控制器，并将底层事件通过nfa_dm_nfc_response_cback回调函数传递给NFA设备管理模块。

`NFC_Enable(nfa_dm_nfc_response_cback)` 是 NFC 协议栈底层的关键启用函数，其核心作用是**初始化并激活 NFC 控制器（NFCC）**，同时建立底层事件上报通道。具体作用如下：

------

### 核心功能

1. **NFCC 硬件初始化**：
   - 发送复位命令（CORE_RESET_CMD）
   - 初始化 NFCC 寄存器
   - 下载 NFCC 固件（如果需要）
   - 配置时钟、RF 参数等硬件设置
2. **建立事件上报通道**：
   - 注册回调函数 `nfa_dm_nfc_response_cback`
   - 所有底层事件（NCI 响应/通知）通过此回调上报给 NFA 层
3. **NCI 协议栈初始化**：
   - 初始化 NCI 传输层（如 I2C/UART）
   - 设置 NCI 数据包解析器
   - 创建 NFCC 控制线程

------

### 工作流程

c



复制



下载

```
// nfa_dm_act.c
void nfa_dm_enable(tNFA_DM_MSG* p_data) {
  // 1. 保存应用层回调
  nfa_dm_cb.p_dm_cback = p_data->enable.p_dm_cback;
  nfa_dm_cb.p_conn_cback = p_data->enable.p_conn_cback;
  
  // 2. 调用底层启用函数
  NFC_Enable(nfa_dm_nfc_response_cback); // <<< 关键调用
}

// nfc_main.c
void NFC_Enable(tNFC_RESPONSE_CBACK* p_cback) {
  nfc_cb.p_resp_cback = p_cback; // 注册回调
  
  // 3. 发送硬件复位命令
  nfc_ncif_send_cmd(&cmd);
}
```

------

### 回调机制

图表

代码







下载

<svg role="graphics-document document" viewBox="-3.5 -3.5 1209.0555419921875 77" class="flowchart mermaid-svg" xmlns="http://www.w3.org/2000/svg" width="100%" id="mermaid-svg-0" height="100%" style="max-width: 100%; transform-origin: 0px 0px; user-select: none; transform: translate(0px, 76.0555px) scale(1);"><g><marker orient="auto" markerHeight="8" markerWidth="8" markerUnits="userSpaceOnUse" refY="5" refX="5" viewBox="0 0 10 10" class="marker flowchart-v2" id="mermaid-svg-0_flowchart-v2-pointEnd"><path style="stroke-width: 1; stroke-dasharray: 1, 0;" class="arrowMarkerPath" d="M 0 0 L 10 5 L 0 10 z"></path></marker><marker orient="auto" markerHeight="8" markerWidth="8" markerUnits="userSpaceOnUse" refY="5" refX="4.5" viewBox="0 0 10 10" class="marker flowchart-v2" id="mermaid-svg-0_flowchart-v2-pointStart"><path style="stroke-width: 1; stroke-dasharray: 1, 0;" class="arrowMarkerPath" d="M 0 5 L 10 10 L 10 0 z"></path></marker><marker orient="auto" markerHeight="11" markerWidth="11" markerUnits="userSpaceOnUse" refY="5" refX="11" viewBox="0 0 10 10" class="marker flowchart-v2" id="mermaid-svg-0_flowchart-v2-circleEnd"><circle style="stroke-width: 1; stroke-dasharray: 1, 0;" class="arrowMarkerPath" r="5" cy="5" cx="5"></circle></marker><marker orient="auto" markerHeight="11" markerWidth="11" markerUnits="userSpaceOnUse" refY="5" refX="-1" viewBox="0 0 10 10" class="marker flowchart-v2" id="mermaid-svg-0_flowchart-v2-circleStart"><circle style="stroke-width: 1; stroke-dasharray: 1, 0;" class="arrowMarkerPath" r="5" cy="5" cx="5"></circle></marker><marker orient="auto" markerHeight="11" markerWidth="11" markerUnits="userSpaceOnUse" refY="5.2" refX="12" viewBox="0 0 11 11" class="marker cross flowchart-v2" id="mermaid-svg-0_flowchart-v2-crossEnd"><path style="stroke-width: 2; stroke-dasharray: 1, 0;" class="arrowMarkerPath" d="M 1,1 l 9,9 M 10,1 l -9,9"></path></marker><marker orient="auto" markerHeight="11" markerWidth="11" markerUnits="userSpaceOnUse" refY="5.2" refX="-1" viewBox="0 0 11 11" class="marker cross flowchart-v2" id="mermaid-svg-0_flowchart-v2-crossStart"><path style="stroke-width: 2; stroke-dasharray: 1, 0;" class="arrowMarkerPath" d="M 1,1 l 9,9 M 10,1 l -9,9"></path></marker><g class="root"><g class="clusters"></g><g class="edgePaths"><path marker-end="url(#mermaid-svg-0_flowchart-v2-pointEnd)" style="" class="edge-thickness-normal edge-pattern-solid edge-thickness-normal edge-pattern-solid flowchart-link" id="L_NFCC_NFC_Layer_0" d="M173.431,35L182.685,35C191.94,35,210.449,35,228.292,35C246.134,35,263.31,35,271.898,35L280.486,35"></path><path marker-end="url(#mermaid-svg-0_flowchart-v2-pointEnd)" style="" class="edge-thickness-normal edge-pattern-solid edge-thickness-normal edge-pattern-solid flowchart-link" id="L_NFC_Layer_NFA_Layer_0" d="M420.403,35L441.236,35C462.069,35,503.736,35,544.736,35C585.736,35,626.069,35,646.236,35L666.403,35"></path><path marker-end="url(#mermaid-svg-0_flowchart-v2-pointEnd)" style="" class="edge-thickness-normal edge-pattern-solid edge-thickness-normal edge-pattern-solid flowchart-link" id="L_NFA_Layer_Application_0" d="M804.5,35L825.204,35C845.907,35,887.315,35,928.056,35C968.796,35,1008.87,35,1028.907,35L1048.944,35"></path></g><g class="edgeLabels"><g transform="translate(228.95833587646484, 35)" class="edgeLabel"><g transform="translate(-30.52777862548828, -12)" class="label"><foreignObject height="24" width="61.05555725097656"><div class="labelBkg" xmlns="http://www.w3.org/1999/xhtml" style="background-color: rgba(88, 88, 88, 0.5); display: table-cell; white-space: nowrap; line-height: 1.5; max-width: 200px; text-align: center;"><span class="edgeLabel" style="fill: rgb(204, 204, 204); color: rgb(204, 204, 204); background-color: rgb(88, 88, 88); text-align: center;"><p style="margin: 0px; background-color: rgb(88, 88, 88);">NCI 事件</p></span></div></foreignObject></g></g><g transform="translate(545.4027862548828, 35)" class="edgeLabel"><g transform="translate(-100, -24)" class="label"><foreignObject height="48" width="200"><div class="labelBkg" xmlns="http://www.w3.org/1999/xhtml" style="background-color: rgba(88, 88, 88, 0.5); display: table; white-space: break-spaces; line-height: 1.5; max-width: 200px; text-align: center; width: 200px;"><span class="edgeLabel" style="fill: rgb(204, 204, 204); color: rgb(204, 204, 204); background-color: rgb(88, 88, 88); text-align: center;"><p style="margin: 0px; background-color: rgb(88, 88, 88);">nfa_dm_nfc_response_cback</p></span></div></foreignObject></g></g><g transform="translate(928.7222366333008, 35)" class="edgeLabel"><g transform="translate(-99.22222137451172, -12)" class="label"><foreignObject height="24" width="198.44444274902344"><div class="labelBkg" xmlns="http://www.w3.org/1999/xhtml" style="background-color: rgba(88, 88, 88, 0.5); display: table-cell; white-space: nowrap; line-height: 1.5; max-width: 200px; text-align: center;"><span class="edgeLabel" style="fill: rgb(204, 204, 204); color: rgb(204, 204, 204); background-color: rgb(88, 88, 88); text-align: center;"><p style="margin: 0px; background-color: rgb(88, 88, 88);">p_dm_cback/p_conn_cback</p></span></div></foreignObject></g></g></g><g class="nodes"><g transform="translate(90.71527862548828, 35)" id="flowchart-NFCC-0" class="node default"><rect height="54" width="165.43055725097656" y="-27" x="-82.71527862548828" style="" class="basic label-container"></rect><g transform="translate(-52.71527862548828, -12)" style="" class="label"><rect></rect><foreignObject height="24" width="105.43055725097656"><div xmlns="http://www.w3.org/1999/xhtml" style="display: table-cell; white-space: nowrap; line-height: 1.5; max-width: 200px; text-align: center;"><span class="nodeLabel" style="fill: rgb(204, 204, 204); color: rgb(204, 204, 204);"><p style="margin: 0px;">NFC Controller</p></span></div></foreignObject></g></g><g transform="translate(352.44445037841797, 35)" id="flowchart-NFC_Layer-1" class="node default"><rect height="54" width="135.91666412353516" y="-27" x="-67.95833206176758" style="" class="basic label-container"></rect><g transform="translate(-37.95833206176758, -12)" style="" class="label"><rect></rect><foreignObject height="24" width="75.91666412353516"><div xmlns="http://www.w3.org/1999/xhtml" style="display: table-cell; white-space: nowrap; line-height: 1.5; max-width: 200px; text-align: center;"><span class="nodeLabel" style="fill: rgb(204, 204, 204); color: rgb(204, 204, 204);"><p style="margin: 0px;">NFC_Layer</p></span></div></foreignObject></g></g><g transform="translate(737.4514007568359, 35)" id="flowchart-NFA_Layer-3" class="node default"><rect height="54" width="134.09722137451172" y="-27" x="-67.04861068725586" style="" class="basic label-container"></rect><g transform="translate(-37.04861068725586, -12)" style="" class="label"><rect></rect><foreignObject height="24" width="74.09722137451172"><div xmlns="http://www.w3.org/1999/xhtml" style="display: table-cell; white-space: nowrap; line-height: 1.5; max-width: 200px; text-align: center;"><span class="nodeLabel" style="fill: rgb(204, 204, 204); color: rgb(204, 204, 204);"><p style="margin: 0px;">NFA_Layer</p></span></div></foreignObject></g></g><g transform="translate(1123.500015258789, 35)" id="flowchart-Application-5" class="node default"><rect height="54" width="141.11111450195312" y="-27" x="-70.55555725097656" style="" class="basic label-container"></rect><g transform="translate(-40.55555725097656, -12)" style="" class="label"><rect></rect><foreignObject height="24" width="81.11111450195312"><div xmlns="http://www.w3.org/1999/xhtml" style="display: table-cell; white-space: nowrap; line-height: 1.5; max-width: 200px; text-align: center;"><span class="nodeLabel" style="fill: rgb(204, 204, 204); color: rgb(204, 204, 204);"><p style="margin: 0px;">Application</p></span></div></foreignObject></g></g></g></g></g></svg>

1. **底层事件流向**：
   - NFCC 产生中断/数据 → NFC 层封装为 NCI 消息 → 通过 `nfa_dm_nfc_response_cback` 上报
2. **回调处理示例**：

c



复制



下载

```
void nfa_dm_nfc_response_cback(tNFC_RESPONSE_EVT event, 
                               tNFC_RESPONSE* p_data) {
  switch(event) {
    case NFC_ENABLE_REVT: // NFCC启用完成
      nfa_dm_handle_enable_event(p_data);
      break;
    case NFC_RF_DISCOVER_REVT: // 发现标签
      nfa_dm_activate_handle_discovery(p_data);
      break;
    // ...其他事件处理
  }
}
```

------

### 关键特性

1. **异步执行**：

   - 函数立即返回，实际初始化在后台线程执行
   - 结果通过 `NFC_ENABLE_REVT` 事件回调通知

2. **错误处理**：

   - 超时检测（通常 1-2 秒）
   - 固件下载失败重试机制
   - 硬件故障上报（`NFC_STATUS_FAILED`）

3. **状态管理**：

   c

   

   复制

   

   下载

   ```
   typedef enum {
     NFC_STATUS_OK,        // 0x00
     NFC_STATUS_REJECTED,  // 0x01
     NFC_STATUS_FAILED     // 0x02
   } tNFC_STATUS;
   ```

   通过回调参数返回操作状态

------

### 与 NFA_Enable() 的关系

c



复制



下载

```
NFA_Enable()
  → 发送 NFA_DM_API_ENABLE_EVT
    → nfa_dm_enable()
      → NFC_Enable(nfa_dm_nfc_response_cback)  // 核心调用
        → 底层初始化完成后触发 NFC_ENABLE_REVT
          → nfa_dm_handle_enable_event()
            → 通知应用层 NFA_DM_ENABLE_EVT
```

**定位**：作为 NFA 启用流程的底层执行引擎，将物理硬件抽象为可用的 NFC 服务。









tNFC_STATUS NFC_Enable(tNFC_RESPONSE_CBACK* p_cback) {  LOG(VERBOSE) << __func__;   /* Validate callback */  if (!p_cback) {    return (NFC_STATUS_INVALID_PARAM);  }  nfc_cb.p_resp_cback = p_cback;   /* Open HAL transport. */  nfc_set_state(NFC_STATE_W4_HAL_OPEN);  nfc_cb.p_hal->open(nfc_main_hal_cback, nfc_main_hal_data_cback);   return (NFC_STATUS_OK); }









已深度思考（用时 28 秒）



我们分析一下提供的`NFC_Enable`函数代码：

1. **函数签名**：

​    `tNFC_STATUS NFC_Enable(tNFC_RESPONSE_CBACK* p_cback)`

2. **参数**：

​    \- `p_cback`: 一个指向回调函数的指针，用于处理NFC层的响应事件。

3. **功能**：

​    这个函数是NFC协议栈初始化的关键一步，它负责打开与NFC控制器的硬件抽象层（HAL）通信。

4. **代码解析**：

​    \- **参数校验**：首先检查传入的回调函数指针`p_cback`是否有效（非空）。如果为空，则返回`NFC_STATUS_INVALID_PARAM`。

​    \- **保存回调函数**：将传入的回调函数指针保存到全局结构体`nfc_cb`的`p_resp_cback`成员中。这样，当NFC层有事件时，可以通过这个回调函数通知上层。

​    \- **设置状态**：调用`nfc_set_state(NFC_STATE_W4_HAL_OPEN)`，将NFC状态设置为等待HAL打开完成的状态。

​    \- **打开HAL传输层**：通过调用`nfc_cb.p_hal->open`来打开HAL传输层。这里传递了两个回调函数：

​        \- `nfc_main_hal_cback`：用于处理HAL层的事件回调。

​        \- `nfc_main_hal_data_cback`：用于处理HAL层的数据回调。

5. **返回值**：

​    如果上述步骤都成功，则返回`NFC_STATUS_OK`。

6. **后续流程**：

​    当HAL层打开完成后，会调用`nfc_main_hal_cback`，在该回调中，NFC层会继续后续的初始化步骤，例如发送复位命令（CORE_RESET_CMD）等。

7. **注意**：

​    这个函数只是启动了初始化过程，真正的初始化完成是通过回调函数异步通知的。当HAL层成功打开后，会触发`HAL_NFC_OPEN_CPLT_EVT`事件，然后NFC层会继续执行后续的初始化操作。

8. **流程图**：

​    调用`NFC_Enable` -> 保存回调 -> 设置状态为`NFC_STATE_W4_HAL_OPEN` -> 调用HAL的`open`函数 -> 等待HAL打开完成 -> 在`nfc_main_hal_cback`中处理打开结果 -> 继续初始化（如发送复位命令等）-> 最终通过回调函数`p_cback`通知上层初始化结果（如`NFC_ENABLE_REVT`事件）。

9. **关键点**：

​    这个函数是异步的，它只是启动了初始化过程，而初始化的结果（成功或失败）会通过传入的回调函数`p_cback`来通知。

10. **错误处理**：

​     如果HAL的`open`操作失败，那么HAL回调函数`nfc_main_hal_cback`会收到一个错误事件，然后NFC层会通过保存的`p_resp_cback`回调函数通知上层。

 因此，`NFC_Enable`的作用是启动NFC控制器的初始化，包括打开底层的硬件传输层，并设置回调函数来接收初始化过程的事件。

这段代码展示了 `NFC_Enable()` 函数的实现细节，它是 NFC 协议栈初始化的核心部分。下面是对代码的逐行解析和关键作用说明：

### 代码解析

c



复制



下载

```
tNFC_STATUS NFC_Enable(tNFC_RESPONSE_CBACK* p_cback) {
  LOG(VERBOSE) << __func__;  // 记录函数名（调试日志）

  /* Validate callback */
  if (!p_cback) {  // 检查回调函数是否有效
    return (NFC_STATUS_INVALID_PARAM);  // 无效参数直接返回
  }
  
  nfc_cb.p_resp_cback = p_cback;  // 保存回调函数到全局控制块

  /* Open HAL transport. */
  nfc_set_state(NFC_STATE_W4_HAL_OPEN);  // 设置状态：等待HAL打开
  nfc_cb.p_hal->open(nfc_main_hal_cback, nfc_main_hal_data_cback);  // 调用HAL层打开函数

  return (NFC_STATUS_OK);  // 返回成功（异步操作）
}
```

### 关键组件说明

1. **回调注册 (`nfc_cb.p_resp_cback = p_cback`)**
   - 保存上层（NFA层）传入的事件回调函数指针
   - 后续所有底层事件（硬件状态、数据接收等）都通过此回调上报
2. **状态管理 (`nfc_set_state(NFC_STATE_W4_HAL_OPEN)`)**
   - 设置协议栈状态为 **等待HAL打开完成**
   - 状态机驱动后续操作流程
3. **HAL层激活 (`p_hal->open()`)**
   - 调用硬件抽象层的打开函数
   - 传递两个核心回调函数：
     - `nfc_main_hal_cback`：处理控制事件（连接状态变化等）
     - `nfc_main_hal_data_cback`：处理数据收发

### 执行流程图

图表

代码







下载

设备驱动硬件抽象层NFC层NFA层应用程序设备驱动硬件抽象层NFC层NFA层应用程序NFA_Enable()NFC_Enable(nfa_dm_nfc_response_cback)p_hal->open(cback, data_cback)打开物理传输层(I2C/UART/SPI)打开结果nfc_main_hal_cback(HAL_NFC_OPEN_CPLT_EVT)nfa_dm_nfc_response_cback(NFC_ENABLE_REVT)p_dm_cback(NFA_DM_ENABLE_EVT)

### 关键设计特点

1. **异步架构**：

   - 函数立即返回 `NFC_STATUS_OK`
   - 实际结果通过回调异步通知
   - 避免阻塞调用线程

2. **分层回调机制**：

   c

   

   复制

   

   下载

   ```
   // 事件回调原型
   typedef void(tNFC_RESPONSE_CBACK)(tNFC_RESPONSE_EVT event, 
                                    tNFC_RESPONSE* p_data);
   
   // HAL事件回调
   void nfc_main_hal_cback(tHAL_NFC_EVT event, tHAL_NFC_STATUS status) {
     if (event == HAL_NFC_OPEN_CPLT_EVT) {
       if (status == HAL_NFC_STATUS_OK) {
         nfc_cb.p_resp_cback(NFC_ENABLE_REVT, &enable_data);
       }
     }
   }
   ```

3. **状态驱动**：

   c

   

   复制

   

   下载

   ```
   // 状态定义
   enum {
     NFC_STATE_NONE,          // 0: 未初始化
     NFC_STATE_W4_HAL_OPEN,   // 1: 等待HAL打开
     NFC_STATE_CORE_INIT,     // 2: NFCC初始化中
     NFC_STATE_IDLE           // 3: 就绪状态
   };
   
   // 状态转换
   void nfc_main_hal_cback(...) {
     switch (nfc_cb.nfc_state) {
       case NFC_STATE_W4_HAL_OPEN:
         if (event == HAL_NFC_OPEN_CPLT_EVT) {
           nfc_set_state(NFC_STATE_CORE_INIT);
           nci_snd_core_reset(...);  // 发送复位命令
         }
     }
   }
   ```

### 后续处理流程

当 HAL 层完成打开操作后：

1. 触发 `nfc_main_hal_cback(HAL_NFC_OPEN_CPLT_EVT)`
2. NFC 层状态机推进到 `NFC_STATE_CORE_INIT`
3. 发送 NCI 复位命令 (`CORE_RESET_CMD`)
4. 初始化完成后通过 `p_resp_cback` 上报 `NFC_ENABLE_REVT`

> **关键点**：此函数只是 NFCC 启用流程的**起点**，实际硬件初始化在 HAL 层完成。这种分层设计使协议栈能适配不同硬件平台（如博通/NXP/STM 等不同芯片方案）。









那hal层是如何处理的呢









已深度思考（用时 38 秒）



HAL层（硬件抽象层）是连接NFC协议栈（NFC层）和底层设备驱动（如I2C、UART、SPI等物理传输层）的桥梁。其核心目标是**屏蔽硬件差异**，为上层提供统一的控制接口。下面以`open()`函数为例，详细说明HAL层如何处理：

\---

\### 1. HAL层接口定义

在`nfc_hal_api.h`中定义了硬件操作接口结构体：

\```c

typedef struct {

  tHAL_NFC_OPEN open;          // 打开设备

  tHAL_NFC_CLOSE close;        // 关闭设备

  tHAL_NFC_WRITE write;        // 发送数据

  tHAL_NFC_IOCTL ioctl;        // 控制命令

  // ...其他函数指针

} tHAL_NFC_ENTRY;

\```

\---

\### 2. HAL层实现示例

以Linux内核的I2C传输为例（实际路径：`/hardware/nxp/nfc/halimpl/`）：

\```c

// hal_nxp.c

static int hal_open(tHAL_NFC_CBACK *p_hal_cback, 

​                   tHAL_NFC_DATA_CBACK *p_data_cback) {

  /* 1. 打开物理设备文件 */

  fd = open("/dev/i2c-nfc", O_RDWR);

  /* 2. 创建读写线程 */

  pthread_create(&rx_thread, NULL, hal_recv_thread, NULL);

  /* 3. 注册回调 */

  hal_cb.p_hal_cback = p_hal_cback;

  hal_cb.p_data_cback = p_data_cback;

  /* 4. 触发回调通知打开完成 */

  p_hal_cback(HAL_NFC_OPEN_CPLT_EVT, HAL_NFC_STATUS_OK);

  return 0;

}

\```

\---

\### 3. 数据传输流程

\```mermaid

sequenceDiagram

​    participant NFC as NFC层

​    participant HAL as HAL层

​    participant Driver as 设备驱动

​    participant NFCC as NFC控制器

​    NFC->>HAL: open(hal_cback, data_cback)

​    HAL->>Driver: open("/dev/i2c-nfc")

​    Driver-->>HAL: 返回文件描述符fd

​    HAL->>NFC: hal_cback(OPEN_CPLT_EVT)

​    loop 数据接收线程

​        Driver->>HAL: 收到NFCC数据(中断/DMA)

​        HAL->>NFC: data_cback(len, data)

​    end

​    NFC->>HAL: write(cmd_data)

​    HAL->>Driver: write(fd, cmd_data)

​    Driver->>NFCC: 通过I2C发送命令

\```

\---

\### 4. 关键处理机制

\#### (1) 异步事件通知

\- **控制事件**：通过`tHAL_NFC_CBACK`回调

  \```c

  // HAL事件类型

  typedef enum {

​    HAL_NFC_OPEN_CPLT_EVT,      // 打开完成

​    HAL_NFC_CLOSE_CPLT_EVT,     // 关闭完成

​    HAL_NFC_POST_INIT_CPLT_EVT, // 初始化后处理

​    HAL_NFC_ERROR_EVT           // 错误事件

  } tHAL_NFC_EVT;

  \```

\#### (2) 数据通道分离

\- **控制通道**：用于发送NCI命令和接收异步事件

\- **数据通道**：专门传输大数据量（如RF数据包）

  \```c

  // 数据回调原型

  typedef void (*tHAL_NFC_DATA_CBACK)(uint16_t data_len, uint8_t* p_data);

  \```

\#### (3) 超时管理

在发送命令后启动定时器：

\```c

static void hal_start_quick_timer(...) {

  // 设置硬件定时器（如Linux的timerfd）

  timerfd_settime(timer_fd, 0, &itimerspec, NULL);

}

\```

\---

\### 5. 平台适配示例

不同平台的HAL实现位置：

| 平台          | 代码路径                          |

|---------------|----------------------------------|

| Android       | hardware/libhardware/modules/nfc |

| Qualcomm      | vendor/qcom/opensource/nfc       |

| NXP           | vendor/nxp/opensource/halimpl    |

\> **关键设计原则**：  

\> 通过HAL层抽象，同一份NFC协议栈（NFA/NFC层）可无缝移植到不同硬件平台，只需实现对应的`tHAL_NFC_ENTRY`接口即可。

HAL 层（硬件抽象层）是连接 NFC 协议栈和底层硬件的关键桥梁，它处理 `open()` 请求的过程体现了硬件操作的封装和平台适配能力。下面详细分析 HAL 层如何处理这个请求：

------

### 1. HAL 层接口定义

在协议栈初始化时，HAL 层接口已注册到全局控制块：

c



复制



下载

```
// nfc_main.c
tHAL_NFC_ENTRY nfc_hal_entry = {
    .open = HAL_NfcOpen,      // 硬件打开函数
    .close = HAL_NfcClose,    // 硬件关闭函数
    .write = HAL_NfcWrite,    // 数据写入函数
    // ...其他硬件操作函数
};

void NFC_Init(tHAL_NFC_ENTRY* p_hal_entry_tbl) {
    nfc_cb.p_hal = p_hal_entry_tbl; // 保存HAL函数指针
}
```

------

### 2. HAL_Open() 实现解析

以 Linux 平台为例的典型实现路径：
`system/nfc/src/halimpl/pn54x/hal_nfc.cc`

c



复制



下载

```
// hal_nfc.cc
void HAL_NfcOpen(tHAL_NFC_CBACK* p_hal_cback,  // 控制事件回调
                tHAL_NFC_DATA_CBACK* p_data_cback) { // 数据接收回调
{
    LOG(INFO) << __func__;
    
    /* 1. 打开物理设备节点 */
    nfc_fd = open("/dev/pn54x", O_RDWR); // 实际硬件设备文件
    
    /* 2. 创建数据接收线程 */
    pthread_create(&thread, NULL, hal_event_listener_thread, NULL);
    
    /* 3. 注册回调函数 */
    hal_cb.p_hal_cback = p_hal_cback;
    hal_cb.p_data_cback = p_data_cback;
    
    /* 4. 通知打开完成 */
    p_hal_cback(HAL_NFC_OPEN_CPLT_EVT, HAL_NFC_STATUS_OK);
}
```

------

### 3. 关键处理流程

#### (1) 物理设备初始化

| 平台    | 设备节点     | 操作方式 |
| :------ | :----------- | :------- |
| Linux   | /dev/pn54x   | open()   |
| Android | /dev/nq-nci  | ioctl()  |
| QCOM    | /dev/nfc-nci | I2C 读写 |
| NXP     | /dev/pn553   | SPI 传输 |

#### (2) 数据通道建立

c



复制



下载

```
// HAL层事件监听线程
static void* hal_event_listener_thread(void* arg) {
    while (1) {
        // 阻塞读取硬件数据
        len = read(nfc_fd, buffer, MAX_BUFFER_SIZE);
        
        // 区分控制消息和数据消息
        if (is_control_message(buffer)) {
            hal_cb.p_hal_cback(parse_event(buffer), HAL_NFC_STATUS_OK);
        } else {
            hal_cb.p_data_cback(len, buffer); // 原始数据上报
        }
    }
}
```

#### (3) 回调触发机制

图表

代码







下载

DriverHAL_LayerNFC_LayerDriverHAL_LayerNFC_Layer硬件事件发生open(hal_cback, data_cback)open("/dev/pn54x")返回文件描述符创建监听线程hal_cback(OPEN_CPLT_EVT)中断/数据到达通过注册的回调上报

------

### 4. 多平台适配策略

HAL 层使用条件编译支持不同硬件：

c



复制



下载

```
// hal_nfc.cc
#if defined(NXP_PN54X)
    #define NFC_DEV_NODE "/dev/pn54x"
#elif defined(QCOM_NFC)
    #define NFC_DEV_NODE "/dev/nfc-nci"
#elif defined(BRCM_NFC)
    #define NFC_DEV_NODE "/dev/bcm2079x"
#endif

// 统一操作接口
status_t write(const void* data, size_t len) {
    #ifdef USE_I2C
        return i2c_write(addr, data, len);
    #elif defined(USE_UART)
        return uart_send(data, len);
    #endif
}
```

------

### 5. 错误处理机制

c



复制



下载

```
void HAL_NfcOpen(...) {
    if ((nfc_fd = open(DEV_NODE, O_RDWR)) < 0) {
        LOG(ERROR) << "Open failed: " << strerror(errno);
        
        // 分级错误处理
        switch(errno) {
            case ENODEV: // 设备不存在
                hal_cb.p_hal_cback(HAL_NFC_OPEN_CPLT_EVT, HAL_NFC_STATUS_NO_DEVICE);
                break;
            case EACCES: // 权限不足
                hal_cb.p_hal_cback(HAL_NFC_OPEN_CPLT_EVT, HAL_NFC_STATUS_PERMISSION_DENIED);
                break;
            default: 
                hal_cb.p_hal_cback(HAL_NFC_OPEN_CPLT_EVT, HAL_NFC_STATUS_FAILED);
        }
        return;
    }
    // ...成功处理
}
```

------

### 6. 与协议栈的交互

当 HAL 层完成打开操作后：

c



复制



下载

```
// nfc_main.c
static void nfc_main_hal_cback(tHAL_NFC_EVT event, tHAL_NFC_STATUS status) {
    switch (event) {
        case HAL_NFC_OPEN_CPLT_EVT:
            if (status == HAL_NFC_STATUS_OK) {
                // 继续初始化流程
                nfc_set_state(NFC_STATE_CORE_INIT);
                nci_snd_core_reset(NCI_RESET_TYPE_KEEP_CONFIG);
            } else {
                // 通知上层初始化失败
                nfc_cb.p_resp_cback(NFC_ENABLE_REVT, &fail_event);
            }
            break;
    }
}
```

------

### 设计要点总结

1. **硬件抽象**：
   - 统一接口屏蔽底层差异（I2C/SPI/UART）
   - 设备节点抽象化（/dev/pn54x）
2. **异步架构**：
   - 独立线程处理硬件中断
   - 回调机制解耦处理流程
3. **多平台支持**：
   - 条件编译适配不同芯片
   - 错误代码标准化转换
4. **资源管理**：
   - 文件描述符生命周期管理
   - 线程安全的数据传递

通过这种设计，HAL 层使 NFC 协议栈完全独立于硬件实现，只需实现 `tHAL_NFC_ENTRY` 接口即可支持新硬件平台。