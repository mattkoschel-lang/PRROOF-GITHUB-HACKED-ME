# PRROOF-GITHUB-HACKED-ME
I GOT MORE
import {createContext, type PropsWithChildren, useContext} from 'react'

import type {ImmersivePlugin} from './copilot-immersive-plugin'

const PluginsContext = createContext<ImmersivePlugin[]>([])

export function ImmersivePluginsProvider({plugins, children}: PropsWithChildren<{plugins: ImmersivePlugin[]}>) {
  return <PluginsContext.Provider value={plugins}>{children}</PluginsContext.Provider>
}

export function usePlugins(): ImmersivePlugin[] {
  return useContext(PluginsContext)
}

export function usePlugin(id: string | undefined): ImmersivePlugin | null {
  return usePlugins().find(p => p.id === id) ?? null
}

try{ PluginsContext.displayName ||= 'PluginsContext' } catch {}
try{ ImmersivePluginsProvider.displayName ||= 'ImmersivePluginsProvider' } catch {}import type {ImmersivePlugin, PluginComponent} from './copilot-immersive-plugin'

/**
 * Find the active component for a plugin based on the current location
 */
export function findActiveComponent(
  plugin: ImmersivePlugin | undefined,
  location: {pathname: string; search: string; hash: string},
): PluginComponent | undefined {
  if (!plugin?.components) return undefined

  return plugin.components.find(component =>
    component.routes.some(route => {
      if (route instanceof RegExp) {
        return route.test(location.pathname)
      } else {
        return route(location)
      }
    }),
  )
}

/**
 * Get the active plugin from the current location
 */
export function getActivePluginFromLocation(
  location: {pathname: string; search?: string; hash?: string},
  plugins: ImmersivePlugin[],
): ImmersivePlugin | undefined {
  const {pathname} = location
  if (!pathname) return

  // Convert location to full location object for component route matching
  const fullLocation = {
    pathname,
    search: location.search || '',
    hash: location.hash || '',
  }

  return plugins.find(plugin => {
    if (plugin.components && plugin.components.length > 0) {
      return shouldActivatePlugin(plugin, fullLocation)
    }

    return false
  })
}

/**
 * Determines if a plugin should be activated for the given location.
 *
 * If the plugin has explicit routes defined, those take precedence (backward compatibility).
 * If the plugin only has components with routes, activation is derived from component routes.
 *
 * @param plugin The plugin to check
 * @param location The current location
 * @returns true if the plugin should be activated
 */
export function shouldActivatePlugin(
  plugin: ImmersivePlugin,
  location: {pathname: string; search: string; hash: string},
): boolean {
  // Derive activation from component routes
  if (plugin.components && plugin.components.length > 0) {
    return plugin.components.some(component =>
      component.routes.some(route => {
        if (route instanceof RegExp) {
          return route.test(location.pathname)
        }
        // Function-based route matcher
        return route(location)
      }),
    )
  }

  return false
}


import {CopilotReferencePreviewDialog} from '@github-ui/copilot-reference-preview/CopilotReferencePreviewDialog'
import {addUrlToHistoryStack} from '@github-ui/history'
import {sendEvent} from '@github-ui/hydro-analytics'
import type {PathFunction} from '@github-ui/paths'
import {type ReactPartialAnchorProps, useOnAnchorClick} from '@github-ui/react-core/react-partial-anchor'
import {relayEnvironmentWithMissingFieldHandlerForNode} from '@github-ui/relay-environment'
// eslint-disable-next-line no-restricted-imports
import {ScreenSize, ScreenSizeProvider, useScreenSize} from '@github-ui/screen-size'
import {verifiedFetch} from '@github-ui/verified-fetch'
import {CopilotIcon} from '@primer/octicons-react'
import {Button, IconButton, Popover} from '@primer/react'
import {Tooltip} from '@primer/react/deprecated'
import {clsx} from 'clsx'
import React, {useCallback, useEffect, useRef, useState} from 'react'
import {RelayEnvironmentProvider} from 'react-relay'

import {Chat} from './components/Chat'
import {ChatPanel} from './components/ChatPanel'
import {Header} from './components/Header'
import {ChatPortalContainer} from './components/PortalContainerUtils'
import {useEntitlement} from './components/quota/EntitlementContext'
import {ThreadListView} from './components/ThreadListView'
import styles from './CopilotChat.module.css'
import {useSavePageTitle} from './hooks/use-save-page-title'
import {copilotChatHeaderButtonID, copilotChatPanelID} from './utils/constants'
import type {AddCopilotChatReferenceEvent, SearchCopilotEvent, SymbolChangedEvent} from './utils/copilot-chat-events'
import {
  subscribeAddCopilotChatReference,
  subscribeOpenCopilotChat,
  subscribeSearchCopilot,
  subscribeSymbolChanged,
} from './utils/copilot-chat-events'
import {isRepository, makeRepositoryReference} from './utils/copilot-chat-helpers'
import {useResizablePanel} from './utils/copilot-chat-hooks'
import type {LoadingStateState} from './utils/copilot-chat-reducer'
import {
  type CopilotChatOrg,
  type CopilotChatPayload,
  type CopilotChatRepo,
  CopilotLicenseType,
  DialogType,
} from './utils/copilot-chat-types'
import {copilotLocalStorage} from './utils/copilot-local-storage'
import {CopilotChatProvider, useChatState} from './utils/CopilotChatContext'
import {useChatManager} from './utils/CopilotChatManagerContext'
import {getSelectedThread} from './utils/get-selected-thread'
import {useDefaultModel} from './utils/models'

export interface CopilotChatProps extends CopilotChatPayload, ReactPartialAnchorProps {
  currentTopic?: CopilotChatRepo
  findFileWorkerPath: string
  ssoOrganizations: CopilotChatOrg[]
  renderPopover?: boolean
  chatIsVisible?: boolean
  chatVisibleSettingPath?: string
  realIp: string
}

const environment = relayEnvironmentWithMissingFieldHandlerForNode()

export function CopilotChat(props: CopilotChatProps) {
  return (
    <RelayEnvironmentProvider environment={environment}>
      <CopilotChatNoRelay {...props} />
    </RelayEnvironmentProvider>
  )
}

// This component is used to render the assistive experience without the relay environment
// This is useful for testing purposes when the relay environment is provided by renderRelay test helper
export function CopilotChatNoRelay(props: CopilotChatProps) {
  const copilotButtonRef = useRef<HTMLButtonElement>(null)
  const inputRef = useRef<HTMLTextAreaElement>(null)
  const selectedThreadID = copilotLocalStorage.selectedThreadID
  const currentReferences = copilotLocalStorage.getCurrentReferences(selectedThreadID) || []

  return (
    <ScreenSizeProvider>
      <CopilotChatProvider
        topic={props.currentTopic}
        workerPath={props.findFileWorkerPath}
        threadId={selectedThreadID}
        refs={currentReferences}
        mode="assistive"
        ssoOrganizations={props.ssoOrganizations}
        chatIsOpen={false}
        chatIsVisible={props.chatIsVisible}
        chatVisibleSettingPath={props.chatVisibleSettingPath}
        realIp={props.realIp}
        copilotChatPayload={props}
      >
        <CopilotHeaderButton
          renderPopover={props.renderPopover || false}
          ref={copilotButtonRef}
          reactPartialAnchor={props.reactPartialAnchor}
          inputRef={inputRef}
        />
        <ChatPortalContainer />
        <ChatPanelWithHeader inputRef={inputRef} />
      </CopilotChatProvider>
    </ScreenSizeProvider>
  )
}

export const dismissNoticePath: PathFunction<{notice: string}> = ({notice}) => `/settings/dismiss-notice/${notice}`

type CopilotHeaderButtonProps = {
  renderPopover: boolean
  inputRef: React.RefObject<HTMLTextAreaElement | null>
} & ReactPartialAnchorProps
const CopilotHeaderButton = React.forwardRef<HTMLButtonElement, CopilotHeaderButtonProps>((props, ref) => {
  const manager = useChatManager()
  const state = useChatState()
  const selectedThread = getSelectedThread(state)
  const {currentTopic, currentView} = state
  const {screenSize} = useScreenSize()
  const {licenseType} = useEntitlement()
  const defaultModel = useDefaultModel(state.availableModels)
  const initialRenderRef = useRef(true)

  useSavePageTitle()

  useEffect(() => {
    return subscribeOpenCopilotChat(
      e => void manager.handleOpenPanelEvent(selectedThread, e, state.chatIsOpen, currentTopic, defaultModel),
    )
  }, [currentTopic, selectedThread, manager, state.chatIsOpen, defaultModel])

  useEffect(() => {
    return subscribeAddCopilotChatReference((e: AddCopilotChatReferenceEvent) => {
      void manager.handleAddReferenceEvent(e)
    })
  }, [selectedThread, manager, state.currentReferences])

  useEffect(() => {
    return subscribeSymbolChanged((e: SymbolChangedEvent) => {
      void manager.handleSymbolChangedEvent(e)
    })
  })

  useEffect(() => {
    return subscribeSearchCopilot((e: SearchCopilotEvent) => {
      void manager.handleSearchCopilotEvent(e, defaultModel)
    })
  }, [manager, defaultModel])

  useEffect(() => {
    const url = new URL(window.location.href, window.location.origin)
    const params = url.searchParams
    const threadId = state.selectedThreadID
    // copilot=1 url param means that we have opened the page with the intention to open the chat
    // with the threadId that was set in the immersive view
    if (params.get('copilot') === '1' && threadId) {
      const thread = state.threads.get(threadId) ?? null
      void manager.openChat(thread, currentView, 'immersive', state.chatVisibleSettingPath)
      url.searchParams.delete('copilot')
      addUrlToHistoryStack(url.toString())
    }
  }, [manager, state.selectedThreadID, state.threads, currentView, state.chatVisibleSettingPath])

  // Run once on initial render to check if we should open chat on page load
  useEffect(() => {
    if (initialRenderRef.current) {
      initialRenderRef.current = false
      if (state.chatIsVisible && !copilotLocalStorage.getCollapsedState() && screenSize > ScreenSize.large) {
        void manager.openChat(selectedThread, state.currentView, 'page load', state.chatVisibleSettingPath)
      }
    }
  }, [manager, screenSize, selectedThread, state.chatIsVisible, state.chatVisibleSettingPath, state.currentView])

  const openChat = useCallback(async () => {
    if (!state.chatIsOpen) {
      if (licenseType === CopilotLicenseType.Unlicensed) {
        if (state.chatVisibleSettingPath) {
          const data = new FormData()
          data.set('copilot_chat_visible', 'true')
          await verifiedFetch(state.chatVisibleSettingPath, {method: 'PUT', body: data})
          copilotLocalStorage.setCollapsedState(false)
        }
        window.location.replace(`${window.location.origin}/github-copilot/signup?return_to=${window.location.pathname}`)
        return
      }
      void manager.openChat(
        selectedThread,
        currentView,
        'header',
        state.chatVisibleSettingPath,
        isRepository(currentTopic) ? makeRepositoryReference(currentTopic) : undefined,
      )
    } else if (props.inputRef.current) {
      props.inputRef.current.focus()
    }
  }, [
    currentTopic,
    currentView,
    licenseType,
    manager,
    props.inputRef,
    selectedThread,
    state.chatIsOpen,
    state.chatVisibleSettingPath,
  ])

  if (props.reactPartialAnchor) {
    return (
      <div className={styles.CopilotChatContainer}>
        <ExternalAnchorListener reactPartialAnchor={props.reactPartialAnchor} />
        {props.renderPopover ? <WelcomePopover renderPopover /> : <></>}
      </div>
    )
  }

  return (
    <div className={clsx('AppHeader-CopilotChatButton', styles.CopilotChatContainer)}>
      <Tooltip aria-label="Chat with Copilot" direction="s">
        <div>
          {/* eslint-disable-next-line primer-react/a11y-remove-disable-tooltip */}
          <IconButton
            unsafeDisableTooltip
            ref={ref}
            id={copilotChatHeaderButtonID}
            icon={CopilotIcon}
            aria-label="Chat with Copilot"
            aria-controls={copilotChatPanelID}
            onClick={openChat}
            className={clsx('AppHeader-button', styles.IconButton)}
            data-testid="copilot-chat-button"
            data-hotkey="Shift+C"
            aria-expanded={false}
          />
        </div>
      </Tooltip>
      <WelcomePopover renderPopover={props.renderPopover} />
    </div>
  )
})

function ExternalAnchorListener({reactPartialAnchor}: ReactPartialAnchorProps) {
  const onClick = useCallback(() => {
    sendEvent('dotcom_chat.activate', {
      target: `GLOBAL_COPILOT_MENU_HEADER_TO_IMMERSIVE`,
      mode: 'global_nav',
    })
  }, [])

  useOnAnchorClick(reactPartialAnchor!, onClick)
  return <></>
}

function WelcomePopover({renderPopover}: {renderPopover: boolean}) {
  const [showPopover, setShowPopover] = useState(renderPopover)

  const handleClosePopover = useCallback(async () => {
    setShowPopover(false)
    await verifiedFetch(dismissNoticePath({notice: 'copilot_chat_new_user_popover'}), {method: 'POST'})
  }, [setShowPopover])

  return (
    <Popover open={showPopover} caret="top-right" className={styles.Popover}>
      <Popover.Content data-testid="copilot-chat-cta-popover" className={styles.Popover_Content}>
        <p>
          You now have access to{' '}
          <a
            href="https://docs.github.com/enterprise-cloud@latest/copilot/github-copilot-enterprise/overview/about-github-copilot-enterprise"
            target="_blank"
            rel="noopener noreferrer"
          >
            Copilot Enterprise
          </a>
          {'. '}
          Use the Copilot icon to get started.
        </p>
        <Button data-testid="dismiss-copilot-chat-cta-popover" onClick={handleClosePopover}>
          Got it!
        </Button>
      </Popover.Content>
    </Popover>
  )
}

CopilotHeaderButton.displayName = 'CopilotHeaderButton'

interface ChatPanelWithHeaderProps {
  inputRef: React.RefObject<HTMLTextAreaElement | null>
}

function ChatPanelWithHeader({inputRef}: ChatPanelWithHeaderProps) {
  const state = useChatState()
  const manager = useChatManager()
  const staffDialogRef = useRef<HTMLDivElement>(null)
  const [showStaffDialog, setShowStaffDialog] = useState<DialogType>(DialogType.None)
  const [hasUnreadMessages, setHasUnreadMessages] = useState(false)
  const prevMessageCount = useRef<number>(state.messages.length)
  const prevMessageLoadingState = useRef<LoadingStateState>(state.messagesLoading.state)
  const {panelWidth, panelHeight, startResize, onResizerKeyDown} = useResizablePanel()

  useEffect(() => {
    if (
      state.messages.length > prevMessageCount.current &&
      !state.chatIsOpen &&
      prevMessageLoadingState.current === 'loaded'
    ) {
      // eslint-disable-next-line react-hooks/set-state-in-effect
      setHasUnreadMessages(true)
    }
    prevMessageCount.current = state.messages.length
    prevMessageLoadingState.current = state.messagesLoading.state
  }, [state.chatIsOpen, state.messages.length, state.messagesLoading.state])

  useEffect(() => {
    if (state.chatIsOpen && hasUnreadMessages) {
      // eslint-disable-next-line react-hooks/set-state-in-effect
      setHasUnreadMessages(false)
    }
  }, [hasUnreadMessages, state.chatIsOpen])

  return (
    <ChatPanel
      staffDialogRef={staffDialogRef}
      handleClose={(usedEscapeKey?: boolean) => {
        setShowStaffDialog(DialogType.None)
        manager.closeChat()
        const entryId = state.entryPointId ?? copilotChatHeaderButtonID

        if (usedEscapeKey && entryId) {
          const entryElement = document.getElementById(entryId)
          const scrollX = window.scrollX
          const scrollY = window.scrollY

          // Persist current scroll position by delaying focus
          setTimeout(() => {
            entryElement?.focus()
            window.scrollTo(scrollX, scrollY)
          }, 0)
        }
      }}
      panelWidth={panelWidth}
      panelHeight={panelHeight}
      startResize={startResize}
      onResizerKeyDown={onResizerKeyDown}
      initialFocusRef={inputRef}
    >
      <Header
        staffDialogRef={staffDialogRef}
        showStaffDialog={showStaffDialog}
        setShowStaffDialog={setShowStaffDialog}
        isResponding={state.messages.length <= 1 && (!!state.streamingMessage || state.isWaitingOnCopilot)}
      />
      {state.chatIsOpen && (
        <div className={styles.chatContentScrollContainer}>
          <div className={styles.chatViewContainer}>
            {state.currentView === 'thread' ? (
              <>
                <Chat key={state.selectedThreadID} inputRef={inputRef} />
                <CopilotReferencePreviewDialog />
              </>
            ) : (
              <ThreadListView />
            )}
          </div>
        </div>
      )}
    </ChatPanel>
  )
}

ChatPanelWithHeader.displayName = 'ChatPanelWithHeader'

try{ CopilotChat.displayName ||= 'CopilotChat' } catch {}
try{ CopilotChatNoRelay.displayName ||= 'CopilotChatNoRelay' } catch {}
try{ ExternalAnchorListener.displayName ||= 'ExternalAnchorListener' } catch {}
try{ WelcomePopover.displayName ||= 'WelcomePopover' } catch {}

// src/timeoutManager.ts
var defaultTimeoutProvider = {
  // We need the wrapper function syntax below instead of direct references to
  // global setTimeout etc.
  //
  // BAD: `setTimeout: setTimeout`
  // GOOD: `setTimeout: (cb, delay) => setTimeout(cb, delay)`
  //
  // If we use direct references here, then anything that wants to spy on or
  // replace the global setTimeout (like tests) won't work since we'll already
  // have a hard reference to the original implementation at the time when this
  // file was imported.
  setTimeout: (callback, delay) => setTimeout(callback, delay),
  clearTimeout: (timeoutId) => clearTimeout(timeoutId),
  setInterval: (callback, delay) => setInterval(callback, delay),
  clearInterval: (intervalId) => clearInterval(intervalId)
};
var TimeoutManager = class {
  // We cannot have TimeoutManager<T> as we must instantiate it with a concrete
  // type at app boot; and if we leave that type, then any new timer provider
  // would need to support ReturnType<typeof setTimeout>, which is infeasible.
  //
  // We settle for type safety for the TimeoutProvider type, and accept that
  // this class is unsafe internally to allow for extension.
  #provider = defaultTimeoutProvider;
  #providerCalled = false;
  setTimeoutProvider(provider) {
    if (process.env.NODE_ENV !== "production") {
      if (this.#providerCalled && provider !== this.#provider) {
        console.error(
          `[timeoutManager]: Switching provider after calls to previous provider might result in unexpected behavior.`,
          { previous: this.#provider, provider }
        );
      }
    }
    this.#provider = provider;
    if (process.env.NODE_ENV !== "production") {
      this.#providerCalled = false;
    }
  }
  setTimeout(callback, delay) {
    if (process.env.NODE_ENV !== "production") {
      this.#providerCalled = true;
    }
    return this.#provider.setTimeout(callback, delay);
  }
  clearTimeout(timeoutId) {
    this.#provider.clearTimeout(timeoutId);
  }
  setInterval(callback, delay) {
    if (process.env.NODE_ENV !== "production") {
      this.#providerCalled = true;
    }
    return this.#provider.setInterval(callback, delay);
  }
  clearInterval(intervalId) {
    this.#provider.clearInterval(intervalId);
  }
};
var timeoutManager = new TimeoutManager();
function systemSetTimeoutZero(callback) {
  setTimeout(callback, 0);
}
export {
  TimeoutManager,
  defaultTimeoutProvider,
  systemSetTimeoutZero,
  timeoutManager
};
//# sourceMappingURL=timeoutManager.js.map"use client";

// src/useMutation.ts
import * as React from "react";
import {
  MutationObserver,
  noop,
  notifyManager,
  shouldThrowError
} from "@tanstack/query-core";
import { useQueryClient } from "./QueryClientProvider.js";
function useMutation(options, queryClient) {
  const client = useQueryClient(queryClient);
  const [observer] = React.useState(
    () => new MutationObserver(
      client,
      options
    )
  );
  React.useEffect(() => {
    observer.setOptions(options);
  }, [observer, options]);
  const result = React.useSyncExternalStore(
    React.useCallback(
      (onStoreChange) => observer.subscribe(notifyManager.batchCalls(onStoreChange)),
      [observer]
    ),
    () => observer.getCurrentResult(),
    () => observer.getCurrentResult()
  );
  const mutate = React.useCallback(
    (variables, mutateOptions) => {
      observer.mutate(variables, mutateOptions).catch(noop);
    },
    [observer]
  );
  if (result.error && shouldThrowError(observer.options.throwOnError, [result.error])) {
    throw result.error;
  }
  return { ...result, mutate, mutateAsync: result.mutate };
}
export {
  useMutation
};
//# sourceMappingURL=useMutation.js.map

// Copyright Joyent, Inc. and other Node contributors.
//
// Permission is hereby granted, free of charge, to any person obtaining a
// copy of this software and associated documentation files (the
// "Software"), to deal in the Software without restriction, including
// without limitation the rights to use, copy, modify, merge, publish,
// distribute, sublicense, and/or sell copies of the Software, and to permit
// persons to whom the Software is furnished to do so, subject to the
// following conditions:
//
// The above copyright notice and this permission notice shall be included
// in all copies or substantial portions of the Software.
//
// THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS
// OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF
// MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN
// NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM,
// DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR
// OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE
// USE OR OTHER DEALINGS IN THE SOFTWARE.

'use strict';

var R = typeof Reflect === 'object' ? Reflect : null
var ReflectApply = R && typeof R.apply === 'function'
  ? R.apply
  : function ReflectApply(target, receiver, args) {
    return Function.prototype.apply.call(target, receiver, args);
  }

var ReflectOwnKeys
if (R && typeof R.ownKeys === 'function') {
  ReflectOwnKeys = R.ownKeys
} else if (Object.getOwnPropertySymbols) {
  ReflectOwnKeys = function ReflectOwnKeys(target) {
    return Object.getOwnPropertyNames(target)
      .concat(Object.getOwnPropertySymbols(target));
  };
} else {
  ReflectOwnKeys = function ReflectOwnKeys(target) {
    return Object.getOwnPropertyNames(target);
  };
}

function ProcessEmitWarning(warning) {
  if (console && console.warn) console.warn(warning);
}

var NumberIsNaN = Number.isNaN || function NumberIsNaN(value) {
  return value !== value;
}

function EventEmitter() {
  EventEmitter.init.call(this);
}
module.exports = EventEmitter;
module.exports.once = once;

// Backwards-compat with node 0.10.x
EventEmitter.EventEmitter = EventEmitter;

EventEmitter.prototype._events = undefined;
EventEmitter.prototype._eventsCount = 0;
EventEmitter.prototype._maxListeners = undefined;

// By default EventEmitters will print a warning if more than 10 listeners are
// added to it. This is a useful default which helps finding memory leaks.
var defaultMaxListeners = 10;

function checkListener(listener) {
  if (typeof listener !== 'function') {
    throw new TypeError('The "listener" argument must be of type Function. Received type ' + typeof listener);
  }
}

Object.defineProperty(EventEmitter, 'defaultMaxListeners', {
  enumerable: true,
  get: function() {
    return defaultMaxListeners;
  },
  set: function(arg) {
    if (typeof arg !== 'number' || arg < 0 || NumberIsNaN(arg)) {
      throw new RangeError('The value of "defaultMaxListeners" is out of range. It must be a non-negative number. Received ' + arg + '.');
    }
    defaultMaxListeners = arg;
  }
});

EventEmitter.init = function() {

  if (this._events === undefined ||
      this._events === Object.getPrototypeOf(this)._events) {
    this._events = Object.create(null);
    this._eventsCount = 0;
  }

  this._maxListeners = this._maxListeners || undefined;
};

// Obviously not all Emitters should be limited to 10. This function allows
// that to be increased. Set to zero for unlimited.
EventEmitter.prototype.setMaxListeners = function setMaxListeners(n) {
  if (typeof n !== 'number' || n < 0 || NumberIsNaN(n)) {
    throw new RangeError('The value of "n" is out of range. It must be a non-negative number. Received ' + n + '.');
  }
  this._maxListeners = n;
  return this;
};

function _getMaxListeners(that) {
  if (that._maxListeners === undefined)
    return EventEmitter.defaultMaxListeners;
  return that._maxListeners;
}

EventEmitter.prototype.getMaxListeners = function getMaxListeners() {
  return _getMaxListeners(this);
};

EventEmitter.prototype.emit = function emit(type) {
  var args = [];
  for (var i = 1; i < arguments.length; i++) args.push(arguments[i]);
  var doError = (type === 'error');

  var events = this._events;
  if (events !== undefined)
    doError = (doError && events.error === undefined);
  else if (!doError)
    return false;

  // If there is no 'error' event listener then throw.
  if (doError) {
    var er;
    if (args.length > 0)
      er = args[0];
    if (er instanceof Error) {
      // Note: The comments on the `throw` lines are intentional, they show
      // up in Node's output if this results in an unhandled exception.
      throw er; // Unhandled 'error' event
    }
    // At least give some kind of context to the user
    var err = new Error('Unhandled error.' + (er ? ' (' + er.message + ')' : ''));
    err.context = er;
    throw err; // Unhandled 'error' event
  }

  var handler = events[type];

  if (handler === undefined)
    return false;

  if (typeof handler === 'function') {
    ReflectApply(handler, this, args);
  } else {
    var len = handler.length;
    var listeners = arrayClone(handler, len);
    for (var i = 0; i < len; ++i)
      ReflectApply(listeners[i], this, args);
  }

  return true;
};

function _addListener(target, type, listener, prepend) {
  var m;
  var events;
  var existing;

  checkListener(listener);

  events = target._events;
  if (events === undefined) {
    events = target._events = Object.create(null);
    target._eventsCount = 0;
  } else {
    // To avoid recursion in the case that type === "newListener"! Before
    // adding it to the listeners, first emit "newListener".
    if (events.newListener !== undefined) {
      target.emit('newListener', type,
                  listener.listener ? listener.listener : listener);

      // Re-assign `events` because a newListener handler could have caused the
      // this._events to be assigned to a new object
      events = target._events;
    }
    existing = events[type];
  }

  if (existing === undefined) {
    // Optimize the case of one listener. Don't need the extra array object.
    existing = events[type] = listener;
    ++target._eventsCount;
  } else {
    if (typeof existing === 'function') {
      // Adding the second element, need to change to array.
      existing = events[type] =
        prepend ? [listener, existing] : [existing, listener];
      // If we've already got an array, just append.
    } else if (prepend) {
      existing.unshift(listener);
    } else {
      existing.push(listener);
    }

    // Check for listener leak
    m = _getMaxListeners(target);
    if (m > 0 && existing.length > m && !existing.warned) {
      existing.warned = true;
      // No error code for this since it is a Warning
      // eslint-disable-next-line no-restricted-syntax
      var w = new Error('Possible EventEmitter memory leak detected. ' +
                          existing.length + ' ' + String(type) + ' listeners ' +
                          'added. Use emitter.setMaxListeners() to ' +
                          'increase limit');
      w.name = 'MaxListenersExceededWarning';
      w.emitter = target;
      w.type = type;
      w.count = existing.length;
      ProcessEmitWarning(w);
    }
  }

  return target;
}

EventEmitter.prototype.addListener = function addListener(type, listener) {
  return _addListener(this, type, listener, false);
};

EventEmitter.prototype.on = EventEmitter.prototype.addListener;

EventEmitter.prototype.prependListener =
    function prependListener(type, listener) {
      return _addListener(this, type, listener, true);
    };

function onceWrapper() {
  if (!this.fired) {
    this.target.removeListener(this.type, this.wrapFn);
    this.fired = true;
    if (arguments.length === 0)
      return this.listener.call(this.target);
    return this.listener.apply(this.target, arguments);
  }
}

function _onceWrap(target, type, listener) {
  var state = { fired: false, wrapFn: undefined, target: target, type: type, listener: listener };
  var wrapped = onceWrapper.bind(state);
  wrapped.listener = listener;
  state.wrapFn = wrapped;
  return wrapped;
}

EventEmitter.prototype.once = function once(type, listener) {
  checkListener(listener);
  this.on(type, _onceWrap(this, type, listener));
  return this;
};

EventEmitter.prototype.prependOnceListener =
    function prependOnceListener(type, listener) {
      checkListener(listener);
      this.prependListener(type, _onceWrap(this, type, listener));
      return this;
    };

// Emits a 'removeListener' event if and only if the listener was removed.
EventEmitter.prototype.removeListener =
    function removeListener(type, listener) {
      var list, events, position, i, originalListener;

      checkListener(listener);

      events = this._events;
      if (events === undefined)
        return this;

      list = events[type];
      if (list === undefined)
        return this;

      if (list === listener || list.listener === listener) {
        if (--this._eventsCount === 0)
          this._events = Object.create(null);
        else {
          delete events[type];
          if (events.removeListener)
            this.emit('removeListener', type, list.listener || listener);
        }
      } else if (typeof list !== 'function') {
        position = -1;

        for (i = list.length - 1; i >= 0; i--) {
          if (list[i] === listener || list[i].listener === listener) {
            originalListener = list[i].listener;
            position = i;
            break;
          }
        }

        if (position < 0)
          return this;

        if (position === 0)
          list.shift();
        else {
          spliceOne(list, position);
        }

        if (list.length === 1)
          events[type] = list[0];

        if (events.removeListener !== undefined)
          this.emit('removeListener', type, originalListener || listener);
      }

      return this;
    };

EventEmitter.prototype.off = EventEmitter.prototype.removeListener;

EventEmitter.prototype.removeAllListeners =
    function removeAllListeners(type) {
      var listeners, events, i;

      events = this._events;
      if (events === undefined)
        return this;

      // not listening for removeListener, no need to emit
      if (events.removeListener === undefined) {
        if (arguments.length === 0) {
          this._events = Object.create(null);
          this._eventsCount = 0;
        } else if (events[type] !== undefined) {
          if (--this._eventsCount === 0)
            this._events = Object.create(null);
          else
            delete events[type];
        }
        return this;
      }

      // emit removeListener for all listeners on all events
      if (arguments.length === 0) {
        var keys = Object.keys(events);
        var key;
        for (i = 0; i < keys.length; ++i) {
          key = keys[i];
          if (key === 'removeListener') continue;
          this.removeAllListeners(key);
        }
        this.removeAllListeners('removeListener');
        this._events = Object.create(null);
        this._eventsCount = 0;
        return this;
      }

      listeners = events[type];

      if (typeof listeners === 'function') {
        this.removeListener(type, listeners);
      } else if (listeners !== undefined) {
        // LIFO order
        for (i = listeners.length - 1; i >= 0; i--) {
          this.removeListener(type, listeners[i]);
        }
      }

      return this;
    };

function _listeners(target, type, unwrap) {
  var events = target._events;

  if (events === undefined)
    return [];

  var evlistener = events[type];
  if (evlistener === undefined)
    return [];

  if (typeof evlistener === 'function')
    return unwrap ? [evlistener.listener || evlistener] : [evlistener];

  return unwrap ?
    unwrapListeners(evlistener) : arrayClone(evlistener, evlistener.length);
}

EventEmitter.prototype.listeners = function listeners(type) {
  return _listeners(this, type, true);
};

EventEmitter.prototype.rawListeners = function rawListeners(type) {
  return _listeners(this, type, false);
};

EventEmitter.listenerCount = function(emitter, type) {
  if (typeof emitter.listenerCount === 'function') {
    return emitter.listenerCount(type);
  } else {
    return listenerCount.call(emitter, type);
  }
};

EventEmitter.prototype.listenerCount = listenerCount;
function listenerCount(type) {
  var events = this._events;

  if (events !== undefined) {
    var evlistener = events[type];

    if (typeof evlistener === 'function') {
      return 1;
    } else if (evlistener !== undefined) {
      return evlistener.length;
    }
  }

  return 0;
}

EventEmitter.prototype.eventNames = function eventNames() {
  return this._eventsCount > 0 ? ReflectOwnKeys(this._events) : [];
};

function arrayClone(arr, n) {
  var copy = new Array(n);
  for (var i = 0; i < n; ++i)
    copy[i] = arr[i];
  return copy;
}

function spliceOne(list, index) {
  for (; index + 1 < list.length; index++)
    list[index] = list[index + 1];
  list.pop();
}

function unwrapListeners(arr) {
  var ret = new Array(arr.length);
  for (var i = 0; i < ret.length; ++i) {
    ret[i] = arr[i].listener || arr[i];
  }
  return ret;
}

function once(emitter, name) {
  return new Promise(function (resolve, reject) {
    function errorListener(err) {
      emitter.removeListener(name, resolver);
      reject(err);
    }

    function resolver() {
      if (typeof emitter.removeListener === 'function') {
        emitter.removeListener('error', errorListener);
      }
      resolve([].slice.call(arguments));
    };

    eventTargetAgnosticAddListener(emitter, name, resolver, { once: true });
    if (name !== 'error') {
      addErrorHandlerIfEventEmitter(emitter, errorListener, { once: true });
    }
  });
}

function addErrorHandlerIfEventEmitter(emitter, handler, flags) {
  if (typeof emitter.on === 'function') {
    eventTargetAgnosticAddListener(emitter, 'error', handler, flags);
  }
}

function eventTargetAgnosticAddListener(emitter, name, listener, flags) {
  if (typeof emitter.on === 'function') {
    if (flags.once) {
      emitter.once(name, listener);
    } else {
      emitter.on(name, listener);
    }
  } else if (typeof emitter.addEventListener === 'function') {
    // EventTarget does not have `error` event semantics like Node
    // EventEmitters, we do not listen for `error` events here.
    emitter.addEventListener(name, function wrapListener(arg) {
      // IE does not have builtin `{ once: true }` support so we
      // have to do it manually.
      if (flags.once) {
        emitter.removeEventListener(name, wrapListener);
      }
      listener(arg);
    });
  } else {
    throw new TypeError('The "emitter" argument must be of type EventEmitter. Received type ' + typeof emitter);
  }
}
'use strict';

var undefined;

var $Object = require('es-object-atoms');

var $Error = require('es-errors');
var $EvalError = require('es-errors/eval');
var $RangeError = require('es-errors/range');
var $ReferenceError = require('es-errors/ref');
var $SyntaxError = require('es-errors/syntax');
var $TypeError = require('es-errors/type');
var $URIError = require('es-errors/uri');

var abs = require('math-intrinsics/abs');
var floor = require('math-intrinsics/floor');
var max = require('math-intrinsics/max');
var min = require('math-intrinsics/min');
var pow = require('math-intrinsics/pow');
var round = require('math-intrinsics/round');
var sign = require('math-intrinsics/sign');

var $Function = Function;

// eslint-disable-next-line consistent-return
var getEvalledConstructor = function (expressionSyntax) {
	try {
		return $Function('"use strict"; return (' + expressionSyntax + ').constructor;')();
	} catch (e) {}
};

var $gOPD = require('gopd');
var $defineProperty = require('es-define-property');

var throwTypeError = function () {
	throw new $TypeError();
};
var ThrowTypeError = $gOPD
	? (function () {
		try {
			// eslint-disable-next-line no-unused-expressions, no-caller, no-restricted-properties
			arguments.callee; // IE 8 does not throw here
			return throwTypeError;
		} catch (calleeThrows) {
			try {
				// IE 8 throws on Object.getOwnPropertyDescriptor(arguments, '')
				return $gOPD(arguments, 'callee').get;
			} catch (gOPDthrows) {
				return throwTypeError;
			}
		}
	}())
	: throwTypeError;

var hasSymbols = require('has-symbols')();

var getProto = require('get-proto');
var $ObjectGPO = require('get-proto/Object.getPrototypeOf');
var $ReflectGPO = require('get-proto/Reflect.getPrototypeOf');

var $apply = require('call-bind-apply-helpers/functionApply');
var $call = require('call-bind-apply-helpers/functionCall');

var needsEval = {};

var TypedArray = typeof Uint8Array === 'undefined' || !getProto ? undefined : getProto(Uint8Array);

var INTRINSICS = {
	__proto__: null,
	'%AggregateError%': typeof AggregateError === 'undefined' ? undefined : AggregateError,
	'%Array%': Array,
	'%ArrayBuffer%': typeof ArrayBuffer === 'undefined' ? undefined : ArrayBuffer,
	'%ArrayIteratorPrototype%': hasSymbols && getProto ? getProto([][Symbol.iterator]()) : undefined,
	'%AsyncFromSyncIteratorPrototype%': undefined,
	'%AsyncFunction%': needsEval,
	'%AsyncGenerator%': needsEval,
	'%AsyncGeneratorFunction%': needsEval,
	'%AsyncIteratorPrototype%': needsEval,
	'%Atomics%': typeof Atomics === 'undefined' ? undefined : Atomics,
	'%BigInt%': typeof BigInt === 'undefined' ? undefined : BigInt,
	'%BigInt64Array%': typeof BigInt64Array === 'undefined' ? undefined : BigInt64Array,
	'%BigUint64Array%': typeof BigUint64Array === 'undefined' ? undefined : BigUint64Array,
	'%Boolean%': Boolean,
	'%DataView%': typeof DataView === 'undefined' ? undefined : DataView,
	'%Date%': Date,
	'%decodeURI%': decodeURI,
	'%decodeURIComponent%': decodeURIComponent,
	'%encodeURI%': encodeURI,
	'%encodeURIComponent%': encodeURIComponent,
	'%Error%': $Error,
	'%eval%': eval, // eslint-disable-line no-eval
	'%EvalError%': $EvalError,
	'%Float16Array%': typeof Float16Array === 'undefined' ? undefined : Float16Array,
	'%Float32Array%': typeof Float32Array === 'undefined' ? undefined : Float32Array,
	'%Float64Array%': typeof Float64Array === 'undefined' ? undefined : Float64Array,
	'%FinalizationRegistry%': typeof FinalizationRegistry === 'undefined' ? undefined : FinalizationRegistry,
	'%Function%': $Function,
	'%GeneratorFunction%': needsEval,
	'%Int8Array%': typeof Int8Array === 'undefined' ? undefined : Int8Array,
	'%Int16Array%': typeof Int16Array === 'undefined' ? undefined : Int16Array,
	'%Int32Array%': typeof Int32Array === 'undefined' ? undefined : Int32Array,
	'%isFinite%': isFinite,
	'%isNaN%': isNaN,
	'%IteratorPrototype%': hasSymbols && getProto ? getProto(getProto([][Symbol.iterator]())) : undefined,
	'%JSON%': typeof JSON === 'object' ? JSON : undefined,
	'%Map%': typeof Map === 'undefined' ? undefined : Map,
	'%MapIteratorPrototype%': typeof Map === 'undefined' || !hasSymbols || !getProto ? undefined : getProto(new Map()[Symbol.iterator]()),
	'%Math%': Math,
	'%Number%': Number,
	'%Object%': $Object,
	'%Object.getOwnPropertyDescriptor%': $gOPD,
	'%parseFloat%': parseFloat,
	'%parseInt%': parseInt,
	'%Promise%': typeof Promise === 'undefined' ? undefined : Promise,
	'%Proxy%': typeof Proxy === 'undefined' ? undefined : Proxy,
	'%RangeError%': $RangeError,
	'%ReferenceError%': $ReferenceError,
	'%Reflect%': typeof Reflect === 'undefined' ? undefined : Reflect,
	'%RegExp%': RegExp,
	'%Set%': typeof Set === 'undefined' ? undefined : Set,
	'%SetIteratorPrototype%': typeof Set === 'undefined' || !hasSymbols || !getProto ? undefined : getProto(new Set()[Symbol.iterator]()),
	'%SharedArrayBuffer%': typeof SharedArrayBuffer === 'undefined' ? undefined : SharedArrayBuffer,
	'%String%': String,
	'%StringIteratorPrototype%': hasSymbols && getProto ? getProto(''[Symbol.iterator]()) : undefined,
	'%Symbol%': hasSymbols ? Symbol : undefined,
	'%SyntaxError%': $SyntaxError,
	'%ThrowTypeError%': ThrowTypeError,
	'%TypedArray%': TypedArray,
	'%TypeError%': $TypeError,
	'%Uint8Array%': typeof Uint8Array === 'undefined' ? undefined : Uint8Array,
	'%Uint8ClampedArray%': typeof Uint8ClampedArray === 'undefined' ? undefined : Uint8ClampedArray,
	'%Uint16Array%': typeof Uint16Array === 'undefined' ? undefined : Uint16Array,
	'%Uint32Array%': typeof Uint32Array === 'undefined' ? undefined : Uint32Array,
	'%URIError%': $URIError,
	'%WeakMap%': typeof WeakMap === 'undefined' ? undefined : WeakMap,
	'%WeakRef%': typeof WeakRef === 'undefined' ? undefined : WeakRef,
	'%WeakSet%': typeof WeakSet === 'undefined' ? undefined : WeakSet,

	'%Function.prototype.call%': $call,
	'%Function.prototype.apply%': $apply,
	'%Object.defineProperty%': $defineProperty,
	'%Object.getPrototypeOf%': $ObjectGPO,
	'%Math.abs%': abs,
	'%Math.floor%': floor,
	'%Math.max%': max,
	'%Math.min%': min,
	'%Math.pow%': pow,
	'%Math.round%': round,
	'%Math.sign%': sign,
	'%Reflect.getPrototypeOf%': $ReflectGPO
};

if (getProto) {
	try {
		null.error; // eslint-disable-line no-unused-expressions
	} catch (e) {
		// https://github.com/tc39/proposal-shadowrealm/pull/384#issuecomment-1364264229
		var errorProto = getProto(getProto(e));
		INTRINSICS['%Error.prototype%'] = errorProto;
	}
}

var doEval = function doEval(name) {
	var value;
	if (name === '%AsyncFunction%') {
		value = getEvalledConstructor('async function () {}');
	} else if (name === '%GeneratorFunction%') {
		value = getEvalledConstructor('function* () {}');
	} else if (name === '%AsyncGeneratorFunction%') {
		value = getEvalledConstructor('async function* () {}');
	} else if (name === '%AsyncGenerator%') {
		var fn = doEval('%AsyncGeneratorFunction%');
		if (fn) {
			value = fn.prototype;
		}
	} else if (name === '%AsyncIteratorPrototype%') {
		var gen = doEval('%AsyncGenerator%');
		if (gen && getProto) {
			value = getProto(gen.prototype);
		}
	}

	INTRINSICS[name] = value;

	return value;
};

var LEGACY_ALIASES = {
	__proto__: null,
	'%ArrayBufferPrototype%': ['ArrayBuffer', 'prototype'],
	'%ArrayPrototype%': ['Array', 'prototype'],
	'%ArrayProto_entries%': ['Array', 'prototype', 'entries'],
	'%ArrayProto_forEach%': ['Array', 'prototype', 'forEach'],
	'%ArrayProto_keys%': ['Array', 'prototype', 'keys'],
	'%ArrayProto_values%': ['Array', 'prototype', 'values'],
	'%AsyncFunctionPrototype%': ['AsyncFunction', 'prototype'],
	'%AsyncGenerator%': ['AsyncGeneratorFunction', 'prototype'],
	'%AsyncGeneratorPrototype%': ['AsyncGeneratorFunction', 'prototype', 'prototype'],
	'%BooleanPrototype%': ['Boolean', 'prototype'],
	'%DataViewPrototype%': ['DataView', 'prototype'],
	'%DatePrototype%': ['Date', 'prototype'],
	'%ErrorPrototype%': ['Error', 'prototype'],
	'%EvalErrorPrototype%': ['EvalError', 'prototype'],
	'%Float32ArrayPrototype%': ['Float32Array', 'prototype'],
	'%Float64ArrayPrototype%': ['Float64Array', 'prototype'],
	'%FunctionPrototype%': ['Function', 'prototype'],
	'%Generator%': ['GeneratorFunction', 'prototype'],
	'%GeneratorPrototype%': ['GeneratorFunction', 'prototype', 'prototype'],
	'%Int8ArrayPrototype%': ['Int8Array', 'prototype'],
	'%Int16ArrayPrototype%': ['Int16Array', 'prototype'],
	'%Int32ArrayPrototype%': ['Int32Array', 'prototype'],
	'%JSONParse%': ['JSON', 'parse'],
	'%JSONStringify%': ['JSON', 'stringify'],
	'%MapPrototype%': ['Map', 'prototype'],
	'%NumberPrototype%': ['Number', 'prototype'],
	'%ObjectPrototype%': ['Object', 'prototype'],
	'%ObjProto_toString%': ['Object', 'prototype', 'toString'],
	'%ObjProto_valueOf%': ['Object', 'prototype', 'valueOf'],
	'%PromisePrototype%': ['Promise', 'prototype'],
	'%PromiseProto_then%': ['Promise', 'prototype', 'then'],
	'%Promise_all%': ['Promise', 'all'],
	'%Promise_reject%': ['Promise', 'reject'],
	'%Promise_resolve%': ['Promise', 'resolve'],
	'%RangeErrorPrototype%': ['RangeError', 'prototype'],
	'%ReferenceErrorPrototype%': ['ReferenceError', 'prototype'],
	'%RegExpPrototype%': ['RegExp', 'prototype'],
	'%SetPrototype%': ['Set', 'prototype'],
	'%SharedArrayBufferPrototype%': ['SharedArrayBuffer', 'prototype'],
	'%StringPrototype%': ['String', 'prototype'],
	'%SymbolPrototype%': ['Symbol', 'prototype'],
	'%SyntaxErrorPrototype%': ['SyntaxError', 'prototype'],
	'%TypedArrayPrototype%': ['TypedArray', 'prototype'],
	'%TypeErrorPrototype%': ['TypeError', 'prototype'],
	'%Uint8ArrayPrototype%': ['Uint8Array', 'prototype'],
	'%Uint8ClampedArrayPrototype%': ['Uint8ClampedArray', 'prototype'],
	'%Uint16ArrayPrototype%': ['Uint16Array', 'prototype'],
	'%Uint32ArrayPrototype%': ['Uint32Array', 'prototype'],
	'%URIErrorPrototype%': ['URIError', 'prototype'],
	'%WeakMapPrototype%': ['WeakMap', 'prototype'],
	'%WeakSetPrototype%': ['WeakSet', 'prototype']
};

var bind = require('function-bind');
var hasOwn = require('hasown');
var $concat = bind.call($call, Array.prototype.concat);
var $spliceApply = bind.call($apply, Array.prototype.splice);
var $replace = bind.call($call, String.prototype.replace);
var $strSlice = bind.call($call, String.prototype.slice);
var $exec = bind.call($call, RegExp.prototype.exec);

/* adapted from https://github.com/lodash/lodash/blob/4.17.15/dist/lodash.js#L6735-L6744 */
var rePropName = /[^%.[\]]+|\[(?:(-?\d+(?:\.\d+)?)|(["'])((?:(?!\2)[^\\]|\\.)*?)\2)\]|(?=(?:\.|\[\])(?:\.|\[\]|%$))/g;
var reEscapeChar = /\\(\\)?/g; /** Used to match backslashes in property paths. */
var stringToPath = function stringToPath(string) {
	var first = $strSlice(string, 0, 1);
	var last = $strSlice(string, -1);
	if (first === '%' && last !== '%') {
		throw new $SyntaxError('invalid intrinsic syntax, expected closing `%`');
	} else if (last === '%' && first !== '%') {
		throw new $SyntaxError('invalid intrinsic syntax, expected opening `%`');
	}
	var result = [];
	$replace(string, rePropName, function (match, number, quote, subString) {
		result[result.length] = quote ? $replace(subString, reEscapeChar, '$1') : number || match;
	});
	return result;
};
/* end adaptation */

var getBaseIntrinsic = function getBaseIntrinsic(name, allowMissing) {
	var intrinsicName = name;
	var alias;
	if (hasOwn(LEGACY_ALIASES, intrinsicName)) {
		alias = LEGACY_ALIASES[intrinsicName];
		intrinsicName = '%' + alias[0] + '%';
	}

	if (hasOwn(INTRINSICS, intrinsicName)) {
		var value = INTRINSICS[intrinsicName];
		if (value === needsEval) {
			value = doEval(intrinsicName);
		}
		if (typeof value === 'undefined' && !allowMissing) {
			throw new $TypeError('intrinsic ' + name + ' exists, but is not available. Please file an issue!');
		}

		return {
			alias: alias,
			name: intrinsicName,
			value: value
		};
	}

	throw new $SyntaxError('intrinsic ' + name + ' does not exist!');
};

module.exports = function GetIntrinsic(name, allowMissing) {
	if (typeof name !== 'string' || name.length === 0) {
		throw new $TypeError('intrinsic name must be a non-empty string');
	}
	if (arguments.length > 1 && typeof allowMissing !== 'boolean') {
		throw new $TypeError('"allowMissing" argument must be a boolean');
	}

	if ($exec(/^%?[^%]*%?$/, name) === null) {
		throw new $SyntaxError('`%` may not be present anywhere but at the beginning and end of the intrinsic name');
	}
	var parts = stringToPath(name);
	var intrinsicBaseName = parts.length > 0 ? parts[0] : '';

	var intrinsic = getBaseIntrinsic('%' + intrinsicBaseName + '%', allowMissing);
	var intrinsicRealName = intrinsic.name;
	var value = intrinsic.value;
	var skipFurtherCaching = false;

	var alias = intrinsic.alias;
	if (alias) {
		intrinsicBaseName = alias[0];
		$spliceApply(parts, $concat([0, 1], alias));
	}

	for (var i = 1, isOwn = true; i < parts.length; i += 1) {
		var part = parts[i];
		var first = $strSlice(part, 0, 1);
		var last = $strSlice(part, -1);
		if (
			(
				(first === '"' || first === "'" || first === '`')
				|| (last === '"' || last === "'" || last === '`')
			)
			&& first !== last
		) {
			throw new $SyntaxError('property names with quotes must have matching quotes');
		}
		if (part === 'constructor' || !isOwn) {
			skipFurtherCaching = true;
		}

		intrinsicBaseName += '.' + part;
		intrinsicRealName = '%' + intrinsicBaseName + '%';

		if (hasOwn(INTRINSICS, intrinsicRealName)) {
			value = INTRINSICS[intrinsicRealName];
		} else if (value != null) {
			if (!(part in value)) {
				if (!allowMissing) {
					throw new $TypeError('base intrinsic for ' + name + ' exists, but the property is not available.');
				}
				return void undefined;
			}
			if ($gOPD && (i + 1) >= parts.length) {
				var desc = $gOPD(value, part);
				isOwn = !!desc;

				// By convention, when a data property is converted to an accessor
				// property to emulate a data property that does not suffer from
				// the override mistake, that accessor's getter is marked with
				// an `originalValue` property. Here, when we detect this, we
				// uphold the illusion by pretending to see that original data
				// property, i.e., returning the value rather than the getter
				// itself.
				if (isOwn && 'get' in desc && !('originalValue' in desc.get)) {
					value = desc.get;
				} else {
					value = value[part];
				}
			} else {
				isOwn = hasOwn(value, part);
				value = value[part];
			}

			if (isOwn && !skipFurtherCaching) {
				INTRINSICS[intrinsicRealName] = value;
			}
		}
	}
	return value;
};


'use strict';

var $defineProperty = require('es-define-property');

var hasPropertyDescriptors = function hasPropertyDescriptors() {
	return !!$defineProperty;
};

hasPropertyDescriptors.hasArrayLengthDefineBug = function hasArrayLengthDefineBug() {
	// node v0.6 has a bug where array lengths can be Set but not Defined
	if (!$defineProperty) {
		return null;
	}
	try {
		return $defineProperty([], 'length', { value: 1 }).length !== 1;
	} catch (e) {
		// In Firefox 4-22, defining length on an array throws an exception.
		return true;
	}
};

module.exports = hasPropertyDescriptors;



import _extends from'@babel/runtime/helpers/esm/extends';var r,B=r||(r={});B.Pop="POP";B.Push="PUSH";B.Replace="REPLACE";var C="production"!==process.env.NODE_ENV?function(b){return Object.freeze(b)}:function(b){return b};function D(b,h){if(!b){"undefined"!==typeof console&&console.warn(h);try{throw Error(h);}catch(e){}}}function E(b){b.preventDefault();b.returnValue=""}
function F(){var b=[];return{get length(){return b.length},push:function(h){b.push(h);return function(){b=b.filter(function(e){return e!==h})}},call:function(h){b.forEach(function(e){return e&&e(h)})}}}function H(){return Math.random().toString(36).substr(2,8)}function I(b){var h=b.pathname;h=void 0===h?"/":h;var e=b.search;e=void 0===e?"":e;b=b.hash;b=void 0===b?"":b;e&&"?"!==e&&(h+="?"===e.charAt(0)?e:"?"+e);b&&"#"!==b&&(h+="#"===b.charAt(0)?b:"#"+b);return h}
function J(b){var h={};if(b){var e=b.indexOf("#");0<=e&&(h.hash=b.substr(e),b=b.substr(0,e));e=b.indexOf("?");0<=e&&(h.search=b.substr(e),b=b.substr(0,e));b&&(h.pathname=b)}return h}
function createBrowserHistory(b){function h(){var c=p.location,a=m.state||{};return[a.idx,C({pathname:c.pathname,search:c.search,hash:c.hash,state:a.usr||null,key:a.key||"default"})]}function e(c){return"string"===typeof c?c:I(c)}function x(c,a){void 0===a&&(a=null);return C(_extends({pathname:q.pathname,hash:"",search:""},"string"===typeof c?J(c):c,{state:a,key:H()}))}function z(c){t=c;c=h();v=c[0];q=c[1];d.call({action:t,location:q})}function A(c,a){function f(){A(c,a)}var l=r.Push,k=x(c,
a);if(!g.length||(g.call({action:l,location:k,retry:f}),!1)){var n=[{usr:k.state,key:k.key,idx:v+1},e(k)];k=n[0];n=n[1];try{m.pushState(k,"",n)}catch(G){p.location.assign(n)}z(l)}}function y(c,a){function f(){y(c,a)}var l=r.Replace,k=x(c,a);g.length&&(g.call({action:l,location:k,retry:f}),1)||(k=[{usr:k.state,key:k.key,idx:v},e(k)],m.replaceState(k[0],"",k[1]),z(l))}function w(c){m.go(c)}void 0===b&&(b={});b=b.window;var p=void 0===b?document.defaultView:b,m=p.history,u=null;p.addEventListener("popstate",
function(){if(u)g.call(u),u=null;else{var c=r.Pop,a=h(),f=a[0];a=a[1];if(g.length)if(null!=f){var l=v-f;l&&(u={action:c,location:a,retry:function(){w(-1*l)}},w(l))}else"production"!==process.env.NODE_ENV?D(!1,"You are trying to block a POP navigation to a location that was not created by the history library. The block will fail silently in production, but in general you should do all navigation with the history library (instead of using window.history.pushState directly) to avoid this situation."):
void 0;else z(c)}});var t=r.Pop;b=h();var v=b[0],q=b[1],d=F(),g=F();null==v&&(v=0,m.replaceState(_extends({},m.state,{idx:v}),""));return{get action(){return t},get location(){return q},createHref:e,push:A,replace:y,go:w,back:function(){w(-1)},forward:function(){w(1)},listen:function(c){return d.push(c)},block:function(c){var a=g.push(c);1===g.length&&p.addEventListener("beforeunload",E);return function(){a();g.length||p.removeEventListener("beforeunload",E)}}}};
function createHashHistory(b){function h(){var a=J(m.location.hash.substr(1)),f=a.pathname,l=a.search;a=a.hash;var k=u.state||{};return[k.idx,C({pathname:void 0===f?"/":f,search:void 0===l?"":l,hash:void 0===a?"":a,state:k.usr||null,key:k.key||"default"})]}function e(){if(t)c.call(t),t=null;else{var a=r.Pop,f=h(),l=f[0];f=f[1];if(c.length)if(null!=l){var k=q-l;k&&(t={action:a,location:f,retry:function(){p(-1*k)}},p(k))}else"production"!==process.env.NODE_ENV?D(!1,"You are trying to block a POP navigation to a location that was not created by the history library. The block will fail silently in production, but in general you should do all navigation with the history library (instead of using window.history.pushState directly) to avoid this situation."):
void 0;else A(a)}}function x(a){var f=document.querySelector("base"),l="";f&&f.getAttribute("href")&&(f=m.location.href,l=f.indexOf("#"),l=-1===l?f:f.slice(0,l));return l+"#"+("string"===typeof a?a:I(a))}function z(a,f){void 0===f&&(f=null);return C(_extends({pathname:d.pathname,hash:"",search:""},"string"===typeof a?J(a):a,{state:f,key:H()}))}function A(a){v=a;a=h();q=a[0];d=a[1];g.call({action:v,location:d})}function y(a,f){function l(){y(a,f)}var k=r.Push,n=z(a,f);"production"!==process.env.NODE_ENV?
D("/"===n.pathname.charAt(0),"Relative pathnames are not supported in hash history.push("+JSON.stringify(a)+")"):void 0;if(!c.length||(c.call({action:k,location:n,retry:l}),!1)){var G=[{usr:n.state,key:n.key,idx:q+1},x(n)];n=G[0];G=G[1];try{u.pushState(n,"",G)}catch(K){m.location.assign(G)}A(k)}}function w(a,f){function l(){w(a,f)}var k=r.Replace,n=z(a,f);"production"!==process.env.NODE_ENV?D("/"===n.pathname.charAt(0),"Relative pathnames are not supported in hash history.replace("+JSON.stringify(a)+
")"):void 0;c.length&&(c.call({action:k,location:n,retry:l}),1)||(n=[{usr:n.state,key:n.key,idx:q},x(n)],u.replaceState(n[0],"",n[1]),A(k))}function p(a){u.go(a)}void 0===b&&(b={});b=b.window;var m=void 0===b?document.defaultView:b,u=m.history,t=null;m.addEventListener("popstate",e);m.addEventListener("hashchange",function(){var a=h()[1];I(a)!==I(d)&&e()});var v=r.Pop;b=h();var q=b[0],d=b[1],g=F(),c=F();null==q&&(q=0,u.replaceState(_extends({},u.state,{idx:q}),""));return{get action(){return v},get location(){return d},
createHref:x,push:y,replace:w,go:p,back:function(){p(-1)},forward:function(){p(1)},listen:function(a){return g.push(a)},block:function(a){var f=c.push(a);1===c.length&&m.addEventListener("beforeunload",E);return function(){f();c.length||m.removeEventListener("beforeunload",E)}}}};
function createMemoryHistory(b){function h(d,g){void 0===g&&(g=null);return C(_extends({pathname:t.pathname,search:"",hash:""},"string"===typeof d?J(d):d,{state:g,key:H()}))}function e(d,g,c){return!q.length||(q.call({action:d,location:g,retry:c}),!1)}function x(d,g){u=d;t=g;v.call({action:u,location:t})}function z(d,g){var c=r.Push,a=h(d,g);"production"!==process.env.NODE_ENV?D("/"===t.pathname.charAt(0),"Relative pathnames are not supported in memory history.push("+JSON.stringify(d)+")"):
void 0;e(c,a,function(){z(d,g)})&&(m+=1,p.splice(m,p.length,a),x(c,a))}function A(d,g){var c=r.Replace,a=h(d,g);"production"!==process.env.NODE_ENV?D("/"===t.pathname.charAt(0),"Relative pathnames are not supported in memory history.replace("+JSON.stringify(d)+")"):void 0;e(c,a,function(){A(d,g)})&&(p[m]=a,x(c,a))}function y(d){var g=Math.min(Math.max(m+d,0),p.length-1),c=r.Pop,a=p[g];e(c,a,function(){y(d)})&&(m=g,x(c,a))}void 0===b&&(b={});var w=b;b=w.initialEntries;w=w.initialIndex;var p=(void 0===
b?["/"]:b).map(function(d){var g=C(_extends({pathname:"/",search:"",hash:"",state:null,key:H()},"string"===typeof d?J(d):d));"production"!==process.env.NODE_ENV?D("/"===g.pathname.charAt(0),"Relative pathnames are not supported in createMemoryHistory({ initialEntries }) (invalid entry: "+JSON.stringify(d)+")"):void 0;return g}),m=Math.min(Math.max(null==w?p.length-1:w,0),p.length-1),u=r.Pop,t=p[m],v=F(),q=F();return{get index(){return m},get action(){return u},get location(){return t},createHref:function(d){return"string"===
typeof d?d:I(d)},push:z,replace:A,go:y,back:function(){y(-1)},forward:function(){y(1)},listen:function(d){return v.push(d)},block:function(d){return q.push(d)}}};export{r as Action,createBrowserHistory,createHashHistory,createMemoryHistory,I as createPath,J as parsePath}
//# sourceMappingURL=index.js.map

'use strict';

var reactIs = require('react-is');

/**
 * Copyright 2015, Yahoo! Inc.
 * Copyrights licensed under the New BSD License. See the accompanying LICENSE file for terms.
 */
var REACT_STATICS = {
  childContextTypes: true,
  contextType: true,
  contextTypes: true,
  defaultProps: true,
  displayName: true,
  getDefaultProps: true,
  getDerivedStateFromError: true,
  getDerivedStateFromProps: true,
  mixins: true,
  propTypes: true,
  type: true
};
var KNOWN_STATICS = {
  name: true,
  length: true,
  prototype: true,
  caller: true,
  callee: true,
  arguments: true,
  arity: true
};
var FORWARD_REF_STATICS = {
  '$$typeof': true,
  render: true,
  defaultProps: true,
  displayName: true,
  propTypes: true
};
var MEMO_STATICS = {
  '$$typeof': true,
  compare: true,
  defaultProps: true,
  displayName: true,
  propTypes: true,
  type: true
};
var TYPE_STATICS = {};
TYPE_STATICS[reactIs.ForwardRef] = FORWARD_REF_STATICS;
TYPE_STATICS[reactIs.Memo] = MEMO_STATICS;

function getStatics(component) {
  // React v16.11 and below
  if (reactIs.isMemo(component)) {
    return MEMO_STATICS;
  } // React v16.12 and above


  return TYPE_STATICS[component['$$typeof']] || REACT_STATICS;
}

var defineProperty = Object.defineProperty;
var getOwnPropertyNames = Object.getOwnPropertyNames;
var getOwnPropertySymbols = Object.getOwnPropertySymbols;
var getOwnPropertyDescriptor = Object.getOwnPropertyDescriptor;
var getPrototypeOf = Object.getPrototypeOf;
var objectPrototype = Object.prototype;
function hoistNonReactStatics(targetComponent, sourceComponent, blacklist) {
  if (typeof sourceComponent !== 'string') {
    // don't hoist over string (html) components
    if (objectPrototype) {
      var inheritedComponent = getPrototypeOf(sourceComponent);

      if (inheritedComponent && inheritedComponent !== objectPrototype) {
        hoistNonReactStatics(targetComponent, inheritedComponent, blacklist);
      }
    }

    var keys = getOwnPropertyNames(sourceComponent);

    if (getOwnPropertySymbols) {
      keys = keys.concat(getOwnPropertySymbols(sourceComponent));
    }

    var targetStatics = getStatics(targetComponent);
    var sourceStatics = getStatics(sourceComponent);

    for (var i = 0; i < keys.length; ++i) {
      var key = keys[i];

      if (!KNOWN_STATICS[key] && !(blacklist && blacklist[key]) && !(sourceStatics && sourceStatics[key]) && !(targetStatics && targetStatics[key])) {
        var descriptor = getOwnPropertyDescriptor(sourceComponent, key);

        try {
          // Avoid failures from read-only properties
          defineProperty(targetComponent, key, descriptor);
        } catch (e) {}
      }
    }
  }

  return targetComponent;
}

module.exports = hoistNonReactStatics;


'use strict';

/** @typedef {`$${import('.').InternalSlot}`} SaltedInternalSlot */
/** @typedef {{ [k in SaltedInternalSlot]?: unknown }} SlotsObject */

var hasOwn = require('hasown');
/** @type {import('side-channel').Channel<object, SlotsObject>} */
var channel = require('side-channel')();

var $TypeError = require('es-errors/type');

/** @type {import('.')} */
var SLOT = {
	assert: function (O, slot) {
		if (!O || (typeof O !== 'object' && typeof O !== 'function')) {
			throw new $TypeError('`O` is not an object');
		}
		if (typeof slot !== 'string') {
			throw new $TypeError('`slot` must be a string');
		}
		channel.assert(O);
		if (!SLOT.has(O, slot)) {
			throw new $TypeError('`' + slot + '` is not present on `O`');
		}
	},
	get: function (O, slot) {
		if (!O || (typeof O !== 'object' && typeof O !== 'function')) {
			throw new $TypeError('`O` is not an object');
		}
		if (typeof slot !== 'string') {
			throw new $TypeError('`slot` must be a string');
		}
		var slots = channel.get(O);
		// eslint-disable-next-line no-extra-parens
		return slots && slots[/** @type {SaltedInternalSlot} */ ('$' + slot)];
	},
	has: function (O, slot) {
		if (!O || (typeof O !== 'object' && typeof O !== 'function')) {
			throw new $TypeError('`O` is not an object');
		}
		if (typeof slot !== 'string') {
			throw new $TypeError('`slot` must be a string');
		}
		var slots = channel.get(O);
		// eslint-disable-next-line no-extra-parens
		return !!slots && hasOwn(slots, /** @type {SaltedInternalSlot} */ ('$' + slot));
	},
	set: function (O, slot, V) {
		if (!O || (typeof O !== 'object' && typeof O !== 'function')) {
			throw new $TypeError('`O` is not an object');
		}
		if (typeof slot !== 'string') {
			throw new $TypeError('`slot` must be a string');
		}
		var slots = channel.get(O);
		if (!slots) {
			slots = {};
			channel.set(O, slots);
		}
		// eslint-disable-next-line no-extra-parens
		slots[/** @type {SaltedInternalSlot} */ ('$' + slot)] = V;
	}
};

if (Object.freeze) {
	Object.freeze(SLOT);
}

module.exports = SLOT;


/*#__PURE__*/
export function isSupported() {
  return (
    typeof HTMLButtonElement !== "undefined" &&
    "command" in HTMLButtonElement.prototype &&
    "source" in ((globalThis.CommandEvent || {}).prototype || {})
  );
}

/*#__PURE__*/
export function isPolyfilled() {
  return !/native code/i.test((globalThis.CommandEvent || {}).toString());
}

export function apply() {
  // XXX: Invoker Buttons used to dispatch 'invoke' events instead of
  // 'command' events. We should ensure to prevent 'invoke' events being
  // fired in those browsers.
  // XXX: https://bugs.chromium.org/p/chromium/issues/detail?id=1523183
  // Chrome will dispatch invoke events even with the flag disabled; so
  // we need to capture those to prevent duplicate events.
  document.addEventListener(
    "invoke",
    (e) => {
      if (e.type == "invoke" && e.isTrusted) {
        e.stopImmediatePropagation();
        e.preventDefault();
      }
    },
    true,
  );
  document.addEventListener(
    "command",
    (e) => {
      if (e.type == "command" && e.isTrusted) {
        e.stopImmediatePropagation();
        e.preventDefault();
      }
    },
    true,
  );

  function enumerate(obj, key, enumerable = true) {
    Object.defineProperty(obj, key, {
      ...Object.getOwnPropertyDescriptor(obj, key),
      enumerable,
    });
  }

  function getRootNode(node) {
    if (node && typeof node.getRootNode === "function") {
      return node.getRootNode();
    }
    if (node && node.parentNode) return getRootNode(node.parentNode);
    return node;
  }

  const ShadowRoot = globalThis.ShadowRoot || function () {};

  const commandEventSourceElements = new WeakMap();
  const commandEventActions = new WeakMap();

  class CommandEvent extends Event {
    constructor(type, invokeEventInit = {}) {
      super(type, invokeEventInit);
      const { source, command } = invokeEventInit;
      if (source != null && !(source instanceof Element)) {
        throw new TypeError(`source must be an element`);
      }
      commandEventSourceElements.set(this, source || null);
      commandEventActions.set(
        this,
        command !== undefined ? String(command) : "",
      );
    }

    get [Symbol.toStringTag]() {
      return "CommandEvent";
    }

    get source() {
      if (!commandEventSourceElements.has(this)) {
        throw new TypeError("illegal invocation");
      }
      const source = commandEventSourceElements.get(this);
      if (!(source instanceof Element)) return null;
      const invokerRoot = getRootNode(source);
      if (invokerRoot !== getRootNode(this.target || document)) {
        return invokerRoot.host;
      }
      return source;
    }

    get command() {
      if (!commandEventActions.has(this)) {
        throw new TypeError("illegal invocation");
      }
      return commandEventActions.get(this);
    }

    get action() {
      throw new Error(
        "CommandEvent#action was renamed to CommandEvent#command",
      );
    }

    get invoker() {
      throw new Error(
        "CommandEvent#invoker was renamed to CommandEvent#source",
      );
    }
  }
  enumerate(CommandEvent.prototype, "source");
  enumerate(CommandEvent.prototype, "command");

  class InvokeEvent extends Event {
    constructor() {
      throw new Error(
        "InvokeEvent has been deprecated, it has been renamed to `CommandEvent`",
      );
    }
  }

  const invokerAssociatedElements = new WeakMap();

  function applyInvokerMixin(ElementClass) {
    Object.defineProperties(ElementClass.prototype, {
      commandForElement: {
        enumerable: true,
        configurable: true,
        set(targetElement) {
          if (this.hasAttribute("invokeaction")) {
            throw new TypeError(
              "Element has deprecated `invokeaction` attribute, replace with `command`",
            );
          } else if (this.hasAttribute("invoketarget")) {
            throw new TypeError(
              "Element has deprecated `invoketarget` attribute, replace with `commandfor`",
            );
          } else if (targetElement === null) {
            this.removeAttribute("commandfor");
            invokerAssociatedElements.delete(this);
          } else if (!(targetElement instanceof Element)) {
            throw new TypeError(`commandForElement must be an element or null`);
          } else {
            this.setAttribute("commandfor", "");
            const targetRootNode = getRootNode(targetElement);
            const thisRootNode = getRootNode(this);
            if (
              thisRootNode === targetRootNode ||
              targetRootNode === this.ownerDocument
            ) {
              invokerAssociatedElements.set(this, targetElement);
            } else {
              invokerAssociatedElements.delete(this);
            }
          }
        },
        get() {
          if (this.localName !== "button") {
            return null;
          }
          if (
            this.hasAttribute("invokeaction") ||
            this.hasAttribute("invoketarget")
          ) {
            console.warn(
              "Element has deprecated `invoketarget` or `invokeaction` attribute, use `commandfor` and `command` instead",
            );
            return null;
          }
          if (this.disabled) {
            return null;
          }
          if (this.form && this.getAttribute("type") !== "button") {
            console.warn(
              "Element with `commandFor` is a form participant. " +
                "It should explicitly set `type=button` in order for `commandFor` to work",
            );
            return null;
          }
          const targetElement = invokerAssociatedElements.get(this);
          if (targetElement) {
            if (targetElement.isConnected) {
              return targetElement;
            } else {
              invokerAssociatedElements.delete(this);
              return null;
            }
          }
          const root = getRootNode(this);
          const idref = this.getAttribute("commandfor");
          if (
            (root instanceof Document || root instanceof ShadowRoot) &&
            idref
          ) {
            return root.getElementById(idref) || null;
          }
          return null;
        },
      },
      command: {
        enumerable: true,
        configurable: true,
        get() {
          const value = this.getAttribute("command") || "";
          if (value.startsWith("--")) return value;
          const valueLower = value.toLowerCase();
          switch (valueLower) {
            case "show-modal":
            case "close":
            case "toggle-popover":
            case "hide-popover":
            case "show-popover":
              return valueLower;
          }
          return "";
        },
        set(value) {
          this.setAttribute("command", value);
        },
      },

      invokeAction: {
        enumerable: false,
        configurable: true,
        get() {
          throw new Error(
            `invokeAction is deprecated. It has been renamed to command`,
          );
        },
        set(value) {
          throw new Error(
            `invokeAction is deprecated. It has been renamed to command`,
          );
        },
      },

      invokeTargetElement: {
        enumerable: false,
        configurable: true,
        get() {
          throw new Error(
            `invokeTargetElement is deprecated. It has been renamed to command`,
          );
        },
        set(value) {
          throw new Error(
            `invokeTargetElement is deprecated. It has been renamed to command`,
          );
        },
      },
    });
  }

  const onHandlers = new WeakMap();
  Object.defineProperties(HTMLElement.prototype, {
    oncommand: {
      enumerable: true,
      configurable: true,
      get() {
        oncommandObserver.takeRecords();
        return onHandlers.get(this) || null;
      },
      set(handler) {
        const existing = onHandlers.get(this) || null;
        if (existing) {
          this.removeEventListener("command", existing);
        }
        onHandlers.set(
          this,
          typeof handler === "object" || typeof handler === "function"
            ? handler
            : null,
        );
        if (typeof handler == "function") {
          this.addEventListener("command", handler);
        }
      },
    },
  });
  function applyOnCommandHandler(els) {
    for (const el of els) {
      el.oncommand = new Function("event", el.getAttribute("oncommand"));
    }
  }
  const oncommandObserver = new MutationObserver((records) => {
    for (const record of records) {
      const { target } = record;
      if (record.type === "childList") {
        applyOnCommandHandler(target.querySelectorAll("[oncommand]"));
      } else {
        applyOnCommandHandler([target]);
      }
    }
  });
  oncommandObserver.observe(document, {
    subtree: true,
    childList: true,
    attributeFilter: ["oncommand"],
  });
  applyOnCommandHandler(document.querySelectorAll("[oncommand]"));

  function handleInvokerActivation(event) {
    if (event.defaultPrevented) return;
    if (event.type !== "click") return;
    const oldInvoker = event.target.closest(
      "button[invoketarget], button[invokeaction], input[invoketarget], input[invokeaction]",
    );
    if (oldInvoker) {
      console.warn(
        "Elements with `invoketarget` or `invokeaction` are deprecated and should be renamed to use `commandfor` and `command` respectively",
      );
      if (oldInvoker.matches("input")) {
        throw new Error("Input elements no longer support `commandfor`");
      }
    }

    const source = event.target.closest("button[commandfor], button[command]");
    if (!source) return;

    if (this.form && this.getAttribute("type") !== "button") {
      event.preventDefault();
      throw new Error(
        "Element with `commandFor` is a form participant. " +
          "It should explicitly set `type=button` in order for `commandFor` to work. " +
          "In order for it to act as a Submit button, it must not have command or commandfor attributes",
      );
    }

    if (source.hasAttribute("command") !== source.hasAttribute("commandfor")) {
      const attr = source.hasAttribute("command") ? "command" : "commandfor";
      const missing = source.hasAttribute("command") ? "commandfor" : "command";
      throw new Error(
        `Element with ${attr} attribute must also have a ${missing} attribute to function.`,
      );
    }

    if (
      source.command !== "show-popover" &&
      source.command !== "hide-popover" &&
      source.command !== "toggle-popover" &&
      source.command !== "show-modal" &&
      source.command !== "close" &&
      !source.command.startsWith("--")
    ) {
      console.warn(
        `"${source.command}" is not a valid command value. Custom commands must begin with --`,
      );
      return;
    }

    const invokee = source.commandForElement;
    if (!invokee) return;
    const invokeEvent = new CommandEvent("command", {
      command: source.command,
      source,
      cancelable: true,
    });
    invokee.dispatchEvent(invokeEvent);
    if (invokeEvent.defaultPrevented)
      return;

    const command = invokeEvent.command.toLowerCase();

    if (invokee.popover) {
      const canShow = !invokee.matches(":popover-open");
      const shouldShow =
        canShow && (command === "toggle-popover" || command === "show-popover");
      const shouldHide = !canShow && command === "hide-popover";

      if (shouldShow) {
        invokee.showPopover({ source });
      } else if (shouldHide) {
        invokee.hidePopover();
      }
    } else if (invokee.localName === "dialog") {
      const canShow = !invokee.hasAttribute("open");
      const shouldShow = canShow && command === "show-modal";
      const shouldHide = !canShow && command === "close";

      if (shouldShow) {
        invokee.showModal();
      } else if (shouldHide) {
        invokee.close();
      }
    }
  }

  function setupInvokeListeners(target) {
    target.addEventListener("click", handleInvokerActivation, true);
  }

  function observeShadowRoots(ElementClass, callback) {
    const attachShadow = ElementClass.prototype.attachShadow;
    ElementClass.prototype.attachShadow = function (init) {
      const shadow = attachShadow.call(this, init);
      callback(shadow);
      return shadow;
    };
    const attachInternals = ElementClass.prototype.attachInternals;
    ElementClass.prototype.attachInternals = function () {
      const internals = attachInternals.call(this);
      if (internals.shadowRoot) callback(internals.shadowRoot);
      return internals;
    };
  }

  applyInvokerMixin(globalThis.HTMLButtonElement || function () {});

  observeShadowRoots(globalThis.HTMLElement || function () {}, (shadow) => {
    setupInvokeListeners(shadow);
    oncommandObserver.observe(shadow, { attributeFilter: ["oncommand"] });
    applyOnCommandHandler(shadow.querySelectorAll("[oncommand]"));
  });

  setupInvokeListeners(document);

  Object.defineProperty(window, "CommandEvent", {
    value: CommandEvent,
    configurable: true,
    writable: true,
  });
  Object.defineProperty(window, "InvokeEvent", {
    value: InvokeEvent,
    configurable: true,
    writable: true,
  });
}


'use strict';

var callBound = require('call-bound');
var hasToStringTag = require('has-tostringtag/shams')();
var hasOwn = require('hasown');
var gOPD = require('gopd');

/** @type {import('.')} */
var fn;

if (hasToStringTag) {
	/** @type {(receiver: ThisParameterType<typeof RegExp.prototype.exec>, ...args: Parameters<typeof RegExp.prototype.exec>) => ReturnType<typeof RegExp.prototype.exec>} */
	var $exec = callBound('RegExp.prototype.exec');
	/** @type {object} */
	var isRegexMarker = {};

	var throwRegexMarker = function () {
		throw isRegexMarker;
	};
	/** @type {{ toString(): never, valueOf(): never, [Symbol.toPrimitive]?(): never }} */
	var badStringifier = {
		toString: throwRegexMarker,
		valueOf: throwRegexMarker
	};

	if (typeof Symbol.toPrimitive === 'symbol') {
		badStringifier[Symbol.toPrimitive] = throwRegexMarker;
	}

	/** @type {import('.')} */
	// @ts-expect-error TS can't figure out that the $exec call always throws
	// eslint-disable-next-line consistent-return
	fn = function isRegex(value) {
		if (!value || typeof value !== 'object') {
			return false;
		}

		// eslint-disable-next-line no-extra-parens
		var descriptor = /** @type {NonNullable<typeof gOPD>} */ (gOPD)(/** @type {{ lastIndex?: unknown }} */ (value), 'lastIndex');
		var hasLastIndexDataProperty = descriptor && hasOwn(descriptor, 'value');
		if (!hasLastIndexDataProperty) {
			return false;
		}

		try {
			// eslint-disable-next-line no-extra-parens
			$exec(value, /** @type {string} */ (/** @type {unknown} */ (badStringifier)));
		} catch (e) {
			return e === isRegexMarker;
		}
	};
} else {
	/** @type {(receiver: ThisParameterType<typeof Object.prototype.toString>, ...args: Parameters<typeof Object.prototype.toString>) => ReturnType<typeof Object.prototype.toString>} */
	var $toString = callBound('Object.prototype.toString');
	/** @const @type {'[object RegExp]'} */
	var regexClass = '[object RegExp]';

	/** @type {import('.')} */
	fn = function isRegex(value) {
		// In older browsers, typeof regex incorrectly returns 'function'
		if (!value || (typeof value !== 'object' && typeof value !== 'function')) {
			return false;
		}

		return $toString(value) === regexClass;
	};
}

module.exports = fn;
