<script lang="ts">
	import { onMount } from 'svelte';
	let PEER_CONN: RTCPeerConnection | null = null;
	let joinSpeaker = async () => {
		IS_SPEAKER = true;
		await makePeerConn();
	};
	let joinListener = async () => {
		IS_SPEAKER = false;
		await makePeerConn();
	};

	function makeid(length: number): string {
		var result = '';
		var characters = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789';
		var charactersLength = characters.length;
		for (var i = 0; i < length; i++) {
			result += characters.charAt(Math.floor(Math.random() * charactersLength));
		}
		return result;
	}
	const ID: string = makeid(20);

	const trackIDs: { [x: string]: boolean } = {};

	let WS: WebSocket | null = null;

	let MS: MediaStream | null = null;

	let SETTING_OFFER = false;
	let isSettingRemoteAnswerPending = false;
	let ignoreOffer = false;
	let IS_SPEAKER = false;

	interface WSMsg {
		ev: string;
		id: string;
		data: string;
	}

	function closeConn(pc: RTCPeerConnection) {}

	function reportError(e: any) {
		console.error(e);
	}

	async function sendToServer(data: WSMsg): Promise<boolean> {
		if (WS == null) {
			console.error('Signalling not ready yet');
			return false;
		}
		if (WS.readyState != WS.OPEN) {
			return false;
		}
		try {
			WS.send(JSON.stringify(data));
		} catch (e) {
			console.error(e);
			return false;
		}
		return true;
	}

	async function handleICECandidateEvent(this: RTCPeerConnection, ev: RTCPeerConnectionIceEvent) {
		if (ev.candidate) {
			sendToServer({
				ev: 'new-ice-candidate',
				id: ID,
				data: JSON.stringify(ev.candidate.toJSON())
			});
		}
	}

	async function handleTrackEvent(this: RTCPeerConnection, ev: RTCTrackEvent) {
		MS?.addTrack(ev.track);
	}

	async function handleICEConnectionStateChangeEvent(this: RTCPeerConnection, ev: Event) {
		console.log('iceConnectionState: ', this.iceConnectionState);
		switch (this.iceConnectionState) {
			case 'closed':
			case 'failed':
				closeConn(this);
				break;
		}
	}

	async function handleICEGatheringStateChangeEvent(this: RTCPeerConnection, ev: Event) {
		console.log(`iceGatheringState: ${this.iceGatheringState}`);
	}

	async function handleSignalingStateChangeEvent(this: RTCPeerConnection, ev: Event) {
		console.log('signallingState: ', this.signalingState);
		switch (this.signalingState) {
			case 'closed':
				closeConn(this);
				break;
		}
	}

	async function handleNegotiationNeededEvent(this: RTCPeerConnection, ev: Event) {
		try {
			SETTING_OFFER = true;
			let ev = IS_SPEAKER ? 'speaker-offer' : 'listener-offer';
			await this.setLocalDescription();
			await sendToServer({
				ev: ev,
				id: ID,
				data: JSON.stringify(this.localDescription?.toJSON())
			});
		} catch (err) {
			console.error(err);
		} finally {
			SETTING_OFFER = false;
		}
	}

	async function handleOnIceCandidate(this: RTCPeerConnection, ev: RTCPeerConnectionIceEvent) {
		const ice_candidate = ev.candidate?.toJSON();
		await sendToServer({
			ev: 'ice-candidate',
			id: ID,
			data: JSON.stringify(ice_candidate)
		});
	}

	async function handleOffer(data: WSMsg) {
		if (PEER_CONN?.signalingState != 'stable') {
			return;
		}
		const sdp = JSON.parse(data.data) as RTCSessionDescription;
		await PEER_CONN?.setRemoteDescription(sdp); // SRD rolls back as needed
		await PEER_CONN?.setLocalDescription(await PEER_CONN?.createAnswer());
		// await PEER_CONN?.setLocalDescription();
		let ev = IS_SPEAKER ? 'speaker-answer' : 'listener-answer';
		await sendToServer({
			ev: ev,
			id: ID,
			data: JSON.stringify(PEER_CONN?.localDescription?.toJSON())
		});
	}

	async function handleAnswer(data: WSMsg) {
		const sdp = JSON.parse(data.data) as RTCSessionDescription;
		isSettingRemoteAnswerPending = true;
		console.log('[handleAnswer] -> setting remote description');
		await PEER_CONN?.setRemoteDescription(sdp);
		console.log('[handleAnswer] -> setting remote description  -> done');
		isSettingRemoteAnswerPending = false;
	}

	async function handleNewIceCandidate(data: WSMsg) {
		if (PEER_CONN == null) return;
		const ice_candidate = JSON.parse(data.data) as RTCIceCandidateInit;
		await PEER_CONN.addIceCandidate(ice_candidate);
	}

	let createPeerConn = async (): Promise<RTCPeerConnection> => {
		let pc = new RTCPeerConnection({
			iceServers: [
				{
					urls: [
						'stun:stun.1.google.com:19302',
						'stun:stun.2.google.com:19302',
						'stun:stun.3.google.com:19302',
						'stun:stun.4.google.com:19302'
					]
				},
				{
					urls: `turn:localhost:3478`,
					username: 'user',
					credential: 'pwd',
					credentialType: 'password'
				}
			]
		});
		pc.ontrack = handleTrackEvent;
		pc.oniceconnectionstatechange = handleICEConnectionStateChangeEvent;
		pc.onicegatheringstatechange = handleICEGatheringStateChangeEvent;
		pc.onsignalingstatechange = handleSignalingStateChangeEvent;
		pc.onnegotiationneeded = handleNegotiationNeededEvent;
		pc.onicecandidate = handleOnIceCandidate;
		return pc;
	};
	let makePeerConn = async () => {
		if (PEER_CONN != null) {
			console.error('A connection already exists');
			return;
		}
		let pc = await createPeerConn();
		if (IS_SPEAKER) {
			let stream = await navigator.mediaDevices.getUserMedia({ audio: true });
			stream.getTracks().forEach((track) => {
				// trackIDs[track.id] = true;
				pc.addTrack(track, stream);
			});
		} else {
			pc.addTransceiver('audio', { direction: 'recvonly' });
		}
		PEER_CONN = pc;
	};

	onMount(async () => {
		MS = new MediaStream();
		const audElem = document.querySelector('#audioElem > audio') as HTMLAudioElement;
		audElem.srcObject = MS;
		const ws = new WebSocket(`ws://${window.location.host.split(':')[0]}:3000/ws`);
		ws.onclose = () => {
			console.log('Signalling server closed');
		};
		ws.onerror = () => {
			console.log('Signalling server down');
		};

		ws.onmessage = (msg: MessageEvent) => {
			const data: WSMsg = JSON.parse(msg.data as string);
			console.log(`Received event: ${data.ev}`);
			switch (data.ev) {
				case 'offer':
					handleOffer(data);
					break;
				case 'answer':
					handleAnswer(data);
					break;
				case 'ice-candidate':
					handleNewIceCandidate(data);
					break;
				default:
					console.log(`we don't handle event "${data.ev}" yet`);
			}
		};
		while (ws.readyState != ws.OPEN) {
			if (ws.readyState != ws.CONNECTING) {
				console.error('websocket connection never opened');
				return;
			} else {
				break;
			}
		}
		WS = ws;
	});
</script>

<div>
	<button id="join-speakr" on:click={joinSpeaker}> Join as Speaker </button>
</div>

<div>
	<button id="join-listn" on:click={joinListener}> Join as Listener </button>
</div>

<div id="activity">
	<div id="speakers" />
	<div id="listeners" />
	<div id="audioElem"><audio autoplay /></div>
</div>