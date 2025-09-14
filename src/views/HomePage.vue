<template>
	<IonPage>
		<IonHeader>
			<IonToolbar>
				<IonTitle>Bluetooth Printer Test</IonTitle>
			</IonToolbar>
		</IonHeader>
		<IonContent class="ion-padding">
			<div v-if="!isNativePlatform" class="notice">
				<p>This test page requires running on a real device (Android/iOS).</p>
			</div>

			<section>
				<h2 class="section-title">
					<IonIcon :icon="bluetooth" /> Bluetooth Status
				</h2>
				<div class="grid">
					<div class="card">
						<h3>Environment</h3>
						<ul>
							<li>Native: <b>{{ isNativePlatform ? 'Yes' : 'No' }}</b></li>
							<li>Classic plugin: <b>{{ hasClassic ? 'Available' : 'Missing' }}</b></li>
							<li>BLE plugin: <b>{{ hasBle ? 'Available' : 'Missing' }}</b></li>
						</ul>
						<IonButton color="medium" @click="checkEnvironment">Refresh</IonButton>
					</div>
					<div class="card">
						<h3>Permissions</h3>
						<IonButton @click="ensurePermissions" :disabled="!isNativePlatform">Check & Request</IonButton>
						<p class="muted">Android 12+: BLUETOOTH_CONNECT, BLUETOOTH_SCAN, and Location.</p>
					</div>
				</div>
			</section>

			<section v-if="hasClassic">
				<h2 class="section-title">Classic (SPP) — bluetoothSerial</h2>
				<div class="actions">
					<IonButton @click="enableClassic" color="primary">Enable</IonButton>
					<IonButton @click="listPairedClassic" color="tertiary">Paired</IonButton>
					<IonButton @click="discoverClassic" color="secondary">Discover</IonButton>
					<IonButton @click="disconnectClassic" color="medium" :disabled="!isConnectedClassic">Disconnect</IonButton>
				</div>
				<div class="grid">
					<div class="card">
						<h3>Devices (Classic)</h3>
						<IonList>
							<IonItem v-for="d in classicDevices" :key="d.id">
								<IonLabel>
									<div class="row-between">
										<span>{{ d.name || 'Unknown' }}</span>
										<code>{{ d.id }}</code>
									</div>
								</IonLabel>
								<IonButton slot="end" size="small" @click="connectClassic(d)">Connect</IonButton>
							</IonItem>
						</IonList>
					</div>
					<div class="card">
						<h3>Print (Classic)</h3>
						<IonTextarea v-model="printText" auto-grow label="Text" label-placement="stacked"></IonTextarea>
									<div class="actions">
										<IonButton :disabled="!isConnectedClassic" @click="printClassic">Print Text</IonButton>
										<IonButton :disabled="!isConnectedClassic" color="dark" @click="printEscPosClassic">Print ESC/POS</IonButton>
										<IonButton :disabled="!isConnectedClassic" color="primary" @click="printFormattedClassic">Print Formatted</IonButton>
									</div>
						<p>Status: <b>{{ isConnectedClassic ? 'Connected' : 'Not connected' }}</b> {{ connectedClassicName ? '(' + connectedClassicName + ')' : '' }}</p>
					</div>
				</div>
			</section>

			<section v-if="hasBle">
				<h2 class="section-title">BLE — cordova-plugin-ble-central</h2>
				<div class="actions">
					<IonButton :disabled="isScanning" @click="scanBle" color="secondary">{{ isScanning ? 'Scanning…' : 'Scan' }}</IonButton>
					<IonButton :disabled="!isConnectedBle" color="medium" @click="disconnectBle">Disconnect</IonButton>
				</div>
				<div class="grid">
					<div class="card">
						<h3>Devices (BLE)</h3>
						<IonList>
							<IonItem v-for="d in bleDevices" :key="d.id">
								<IonLabel>
									<div class="row-between">
										<span>{{ d.name || 'Unknown' }}</span>
										<code>{{ d.id }}</code>
									</div>
									<small v-if="typeof d.rssi === 'number'">RSSI: {{ d.rssi }}</small>
								</IonLabel>
								<IonButton slot="end" size="small" @click="connectBle(d)">Connect</IonButton>
							</IonItem>
						</IonList>
					</div>
					<div class="card">
						<h3>Print (BLE)</h3>
						<div class="kv">
							<label>Service UUID</label>
							<input v-model="serviceUUID" />
						</div>
						<div class="kv">
							<label>Write Char UUID</label>
							<input v-model="writeCharacteristicUUID" />
						</div>
						<IonTextarea v-model="printText" auto-grow label="Text" label-placement="stacked"></IonTextarea>
									<div class="actions">
										<IonButton :disabled="!isConnectedBle" @click="printBle">Print Text</IonButton>
										<IonButton :disabled="!isConnectedBle" color="dark" @click="printEscPosBle">Print ESC/POS</IonButton>
										<IonButton :disabled="!isConnectedBle" color="primary" @click="printFormattedBle">Print Formatted</IonButton>
									</div>
						<p>Status: <b>{{ isConnectedBle ? 'Connected' : 'Not connected' }}</b> {{ connectedBleName ? '(' + connectedBleName + ')' : '' }}</p>
					</div>
				</div>
			</section>

			<section>
				<h2 class="section-title">Logs</h2>
				<div class="logs">
					<div v-for="(l, i) in logs" :key="i" class="log-line">{{ l }}</div>
				</div>
				<div class="actions">
					<IonButton color="medium" @click="clearLogs">Clear Logs</IonButton>
				</div>
			</section>
		</IonContent>
	</IonPage>
</template>

<script lang="ts">
import { defineComponent, ref, onMounted } from 'vue';
import { IonPage, IonHeader, IonToolbar, IonTitle, IonContent, IonButton, IonList, IonItem, IonLabel, IonTextarea, IonIcon, toastController } from '@ionic/vue';
import { bluetooth } from 'ionicons/icons';
import { Capacitor } from '@capacitor/core';

type ClassicDevice = { id: string; name: string };
type BleDevice = { id: string; name?: string; rssi?: number; advertising?: any };

export default defineComponent({
	name: 'TestPage',
	components: { IonPage, IonHeader, IonToolbar, IonTitle, IonContent, IonButton, IonList, IonItem, IonLabel, IonTextarea, IonIcon },
	setup() {
		// Environment
		const isNativePlatform = ref(Capacitor.getPlatform() !== 'web');
		const hasClassic = ref(false);
		const hasBle = ref(false);

		// Logs
		const logs = ref<string[]>([]);
		const addLog = (m: string) => {
			const t = new Date().toLocaleTimeString();
			logs.value.unshift(`[${t}] ${m}`);
			// Also console for native logcat
			// eslint-disable-next-line no-console
			console.log(m);
		};
		const clearLogs = () => (logs.value = []);

		// UI shared
		const printText = ref('Hello from Your Shop!\nThis is a print test.\n\n');

		// Classic state
		const classicDevices = ref<ClassicDevice[]>([]);
		const isConnectedClassic = ref(false);
		const connectedClassicName = ref('');

		// BLE state
		const bleDevices = ref<BleDevice[]>([]);
		const isScanning = ref(false);
		const isConnectedBle = ref(false);
		const connectedBleId = ref('');
		const connectedBleName = ref('');
		// Defaults commonly used by BLE serial printers (e.g., HM-10 clones)
		const serviceUUID = ref('49535343-FE7D-4AE5-8FA9-9FAFD205E455');
		const writeCharacteristicUUID = ref('49535343-8841-43F4-A8D4-ECBE34729BB3');

		const showToast = async (message: string, color: 'success' | 'warning' | 'danger' | 'medium' = 'medium') => {
			const t = await toastController.create({ message, duration: 1800, color });
			await t.present();
		};

		const checkEnvironment = () => {
			const w = window as any;
			hasClassic.value = !!w.bluetoothSerial; // cordova-plugin-bluetooth-serial
			hasBle.value = !!w.ble; // cordova-plugin-ble-central
			addLog(`Env -> Native: ${isNativePlatform.value}, Classic: ${hasClassic.value}, BLE: ${hasBle.value}`);
		};

		const ensurePermissions = async () => {
			if (!isNativePlatform.value) return;
			const w = window as any;
			const perms = w.cordova?.plugins?.permissions;
			if (!perms) {
				addLog('Permissions plugin not available; skipping explicit permission requests.');
				return;
			}
			const required = [
				'android.permission.BLUETOOTH_CONNECT',
				'android.permission.BLUETOOTH_SCAN',
				'android.permission.ACCESS_FINE_LOCATION', // Older Android for scanning
			];
			for (const p of required) {
						const has = await new Promise<boolean>((resolve) => {
							// cordova context at runtime
							perms.hasPermission(p, (s: any) => resolve(!!s?.hasPermission), () => resolve(false));
						});
				if (!has) {
					addLog(`Requesting permission: ${p}`);
								const granted = await new Promise<boolean>((resolve) => {
									// cordova context at runtime
									perms.requestPermission(p, (s: any) => resolve(!!s?.hasPermission), () => resolve(false));
								});
					addLog(granted ? `Granted: ${p}` : `Denied: ${p}`);
				} else {
					addLog(`Already granted: ${p}`);
				}
			}
			await showToast('Permission check complete', 'success');
		};

		// Classic helpers
		const enableClassic = async () => {
			try {
				const w = window as any;
				if (!w.bluetoothSerial) return addLog('Classic plugin missing.');
				await new Promise<void>((resolve, reject) => w.bluetoothSerial.enable(resolve, reject));
				addLog('Classic Bluetooth enabled');
				await showToast('Bluetooth enabled', 'success');
			} catch (e) {
				addLog(`Enable failed: ${e}`);
				await showToast('Enable failed', 'danger');
			}
		};

		const listPairedClassic = async () => {
			try {
				const w = window as any;
				if (!w.bluetoothSerial) return addLog('Classic plugin missing.');
				const list = await new Promise<any[]>((resolve, reject) => w.bluetoothSerial.list(resolve, reject));
				classicDevices.value = list.map((d) => ({ id: d.address || d.id, name: d.name || 'Unknown' }));
				addLog(`Paired devices: ${classicDevices.value.length}`);
			} catch (e) {
				addLog(`List paired failed: ${e}`);
			}
		};

		const discoverClassic = async () => {
			try {
				const w = window as any;
				if (!w.bluetoothSerial) return addLog('Classic plugin missing.');
				const list = await new Promise<any[]>((resolve, reject) => w.bluetoothSerial.discoverUnpaired(resolve, reject));
				// merge with paired set
				const merged: Record<string, ClassicDevice> = {};
				for (const d of classicDevices.value) merged[d.id] = d;
				for (const d of list) {
					const id = d.address || d.id;
					merged[id] = { id, name: d.name || 'Unknown' };
				}
				classicDevices.value = Object.values(merged);
				addLog(`Discovered devices: ${list.length}`);
			} catch (e) {
				addLog(`Discover failed: ${e}`);
			}
		};

		const connectClassic = async (d: ClassicDevice) => {
			try {
				const w = window as any;
				if (!w.bluetoothSerial) return addLog('Classic plugin missing.');
				// Try insecure first (better for printers)
				addLog(`Connecting (insecure) to ${d.name} (${d.id})...`);
				try {
					await new Promise<void>((resolve, reject) => w.bluetoothSerial.connectInsecure(d.id, resolve, reject));
				} catch (err) {
					addLog(`Insecure failed: ${err}. Trying standard connect...`);
					await new Promise<void>((resolve, reject) => w.bluetoothSerial.connect(d.id, resolve, reject));
				}
				// Verify
				const ok = await new Promise<boolean>((resolve) => w.bluetoothSerial.isConnected(() => resolve(true), () => resolve(false)));
				isConnectedClassic.value = ok;
				connectedClassicName.value = ok ? (d.name || d.id) : '';
				addLog(ok ? `Connected to ${connectedClassicName.value}` : 'Connect reported success but not connected');
				if (ok) await showToast(`Connected: ${connectedClassicName.value}`, 'success');
			} catch (e) {
				isConnectedClassic.value = false;
				connectedClassicName.value = '';
				addLog(`Connect failed: ${e}`);
				await showToast('Connect failed', 'danger');
			}
		};

		const disconnectClassic = async () => {
			try {
				const w = window as any;
				if (!w.bluetoothSerial) return;
				await new Promise<void>((resolve, reject) => w.bluetoothSerial.disconnect(resolve, reject));
				isConnectedClassic.value = false;
				connectedClassicName.value = '';
				addLog('Classic disconnected');
			} catch (e) {
				addLog(`Disconnect failed: ${e}`);
			}
		};

		const printClassic = async () => {
			try {
				const w = window as any;
				if (!w.bluetoothSerial) return addLog('Classic plugin missing.');
				const text = printText.value;
				await new Promise<void>((resolve, reject) => w.bluetoothSerial.write(text, resolve, reject));
				addLog('Classic: text sent');
				await showToast('Printed (text)', 'success');
			} catch (e) {
				addLog(`Print failed: ${e}`);
				await showToast('Print failed', 'danger');
			}
		};

		const printEscPosClassic = async () => {
			try {
				const w = window as any;
				if (!w.bluetoothSerial) return addLog('Classic plugin missing.');
				const escInit = new Uint8Array([0x1B, 0x40]); // ESC @
				const content = new TextEncoder().encode(printText.value + '\n\n');
				const cut = new Uint8Array([0x1D, 0x56, 0x00]); // GS V 0 (partial/full depends on printer)
				const payload = new Uint8Array(escInit.length + content.length + cut.length);
				payload.set(escInit, 0);
				payload.set(content, escInit.length);
				payload.set(cut, escInit.length + content.length);
				await new Promise<void>((resolve, reject) => w.bluetoothSerial.write(payload.buffer, resolve, reject));
				addLog('Classic: ESC/POS bytes sent');
				await showToast('Printed (ESC/POS)', 'success');
			} catch (e) {
				addLog(`ESC/POS failed: ${e}`);
				await showToast('ESC/POS failed', 'danger');
			}
		};

		// BLE helpers
		const scanBle = async () => {
			const w = window as any;
			if (!w.ble) return addLog('BLE plugin missing.');
			try {
				bleDevices.value = [];
				isScanning.value = true;
				addLog('BLE: scanning for 5 seconds...');
				await new Promise<void>((resolve, reject) => {
					// Scan for all services ([]) for 5 seconds
					w.ble.scan([], 5,
						(device: any) => {
							const existing = bleDevices.value.find((x) => x.id === device.id);
							if (!existing) bleDevices.value.push(device);
						},
						(err: any) => reject(err)
					);
					setTimeout(() => resolve(), 5200);
				});
				addLog(`BLE: found ${bleDevices.value.length} devices`);
			} catch (e) {
				addLog(`BLE scan failed: ${e}`);
			} finally {
				isScanning.value = false;
			}
		};

		const connectBle = async (d: BleDevice) => {
			const w = window as any;
			if (!w.ble) return addLog('BLE plugin missing.');
			try {
				addLog(`BLE: connecting to ${d.name || 'Unknown'} (${d.id})...`);
			await new Promise((resolve, reject) => w.ble.connect(d.id, resolve, reject));
				isConnectedBle.value = true;
				connectedBleId.value = d.id;
				connectedBleName.value = d.name || d.id;
				addLog(`BLE: connected to ${connectedBleName.value}`);
				// Optional: start notification to keep alive if printer supports notify on write char
				// No-op by default since many printers do not notify
				await showToast(`BLE connected: ${connectedBleName.value}`, 'success');
			} catch (e) {
				isConnectedBle.value = false;
				connectedBleId.value = '';
				connectedBleName.value = '';
				addLog(`BLE connect failed: ${e}`);
				await showToast('BLE connect failed', 'danger');
			}
		};

		const disconnectBle = async () => {
			const w = window as any;
			if (!w.ble) return;
			try {
				if (connectedBleId.value) {
					await new Promise<void>((resolve, reject) => w.ble.disconnect(connectedBleId.value, resolve, reject));
				}
				isConnectedBle.value = false;
				connectedBleId.value = '';
				connectedBleName.value = '';
				addLog('BLE: disconnected');
			} catch (e) {
				addLog(`BLE disconnect failed: ${e}`);
			}
		};

		const printBle = async () => {
			const w = window as any;
			if (!w.ble) return addLog('BLE plugin missing.');
			if (!connectedBleId.value) return addLog('BLE: not connected');
			try {
				const data = new TextEncoder().encode(printText.value + '\n');
				await new Promise<void>((resolve, reject) => w.ble.write(connectedBleId.value, serviceUUID.value, writeCharacteristicUUID.value, data.buffer, resolve, reject));
				addLog('BLE: text sent');
				await showToast('BLE printed (text)', 'success');
			} catch (e) {
				addLog(`BLE print failed: ${e}`);
				await showToast('BLE print failed', 'danger');
			}
		};

		const printEscPosBle = async () => {
			const w = window as any;
			if (!w.ble) return addLog('BLE plugin missing.');
			if (!connectedBleId.value) return addLog('BLE: not connected');
			try {
				const escInit = new Uint8Array([0x1B, 0x40]);
				const text = new TextEncoder().encode(printText.value + '\n\n');
				const cut = new Uint8Array([0x1D, 0x56, 0x00]);
				const payload = new Uint8Array(escInit.length + text.length + cut.length);
				payload.set(escInit, 0);
				payload.set(text, escInit.length);
				payload.set(cut, escInit.length + text.length);
				await new Promise<void>((resolve, reject) => w.ble.write(connectedBleId.value, serviceUUID.value, writeCharacteristicUUID.value, payload.buffer, resolve, reject));
				addLog('BLE: ESC/POS bytes sent');
				await showToast('BLE printed (ESC/POS)', 'success');
			} catch (e) {
				addLog(`BLE ESC/POS failed: ${e}`);
				await showToast('BLE ESC/POS failed', 'danger');
			}
		};

			// ===== Formatted ESC/POS builder for 58mm receipt =====
			const buildFormattedReceipt = (title = 'Your Shop Name') => {
				// Commands
				const ESC = 0x1B, GS = 0x1D;
				const init = [ESC, 0x40]; // ESC @
				const alignCenter = [ESC, 0x61, 0x01];
				const alignLeft = [ESC, 0x61, 0x00];
				const boldOn = [ESC, 0x45, 0x01];
				const boldOff = [ESC, 0x45, 0x00];
				const size1x = [GS, 0x21, 0x00];
				const size2x = [GS, 0x21, 0x11]; // double width & height
				const feed1 = [0x0A];
				const cut = [GS, 0x56, 0x00]; // may be partial depending on printer

				const lines: number[] = [];
				const pushTxt = (s: string) => {
					const bytes = Array.from(new TextEncoder().encode(s));
					lines.push(...bytes);
				};

				// Header
				lines.push(...init, ...alignCenter, ...boldOn, ...size2x);
				pushTxt(`${title}\n`);
				lines.push(...size1x, ...boldOff);
				pushTxt('==============================\n');

				// Body
				lines.push(...alignLeft);
				const now = new Date();
				pushTxt(`Date: ${now.toLocaleDateString()}\n`);
				pushTxt(`Time:   ${now.toLocaleTimeString()}\n`);
				pushTxt('------------------------------\n');
				pushTxt('ITEMS\n');
				pushTxt('1 x Haircut               50,000\n');
				pushTxt('1 x Hair Wash             15,000\n');
				pushTxt('------------------------------\n');
				pushTxt('SUBTOTAL:                 65.000\n');
				pushTxt('TAX (10%):                 6,500\n');
				pushTxt('TOTAL:                    71,500\n');
				pushTxt('------------------------------\n');
				pushTxt('CASH:                     75,000\n');
				pushTxt('CHANGE:                    3,500\n');

				// Footer
				lines.push(...alignCenter, ...feed1);
				pushTxt('Thank you!\n\n\n\n');
				lines.push(...feed1, ...cut);

				return new Uint8Array(lines);
			};

      const printFormattedClassic = async () => {
        try {
          const w = window as any;
          if (!w.bluetoothSerial) return addLog('Classic plugin missing.');
          const bytes = buildFormattedReceipt('Your Shop Name');
          await new Promise<void>((resolve, reject) => w.bluetoothSerial.write(bytes.buffer, resolve, reject));
          addLog('Classic: formatted receipt sent');
          await showToast('Printed (formatted)', 'success');
        } catch (e) {
          addLog(`Formatted print failed: ${e}`);
          await showToast('Formatted print failed', 'danger');
        }
      };

      const printFormattedBle = async () => {
        const w = window as any;
        if (!w.ble) return addLog('BLE plugin missing.');
        if (!connectedBleId.value) return addLog('BLE: not connected');
        try {
          const bytes = buildFormattedReceipt('Your Shop Name');
          await new Promise<void>((resolve, reject) => w.ble.write(connectedBleId.value, serviceUUID.value, writeCharacteristicUUID.value, bytes.buffer, resolve, reject));
          addLog('BLE: formatted receipt sent');
          await showToast('BLE printed (formatted)', 'success');
        } catch (e) {
          addLog(`BLE formatted print failed: ${e}`);
          await showToast('BLE formatted print failed', 'danger');
        }
			};

		onMounted(() => {
			checkEnvironment();
			addLog('Bluetooth Printer Test initialized');
		});

		return {
			// icons
			bluetooth,
			// env
			isNativePlatform, hasClassic, hasBle, checkEnvironment, ensurePermissions,
			// logs
			logs, clearLogs,
			// shared
			printText,
			// classic
			classicDevices, isConnectedClassic, connectedClassicName,
			enableClassic, listPairedClassic, discoverClassic, connectClassic, disconnectClassic, printClassic, printEscPosClassic,
			// ble
			bleDevices, isScanning, isConnectedBle, connectedBleName,
			serviceUUID, writeCharacteristicUUID,
			scanBle, connectBle, disconnectBle, printBle, printEscPosBle,
			printFormattedClassic, printFormattedBle,
		};
	},
});
</script>

<style scoped>
.notice { background: #fffbe6; border: 1px solid #ffe58f; padding: 12px; border-radius: 8px; margin-bottom: 12px; }
.section-title { display: flex; align-items: center; gap: 8px; font-size: 18px; margin: 16px 0 8px; }
.grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(280px, 1fr)); gap: 12px; }
.card { border: 1px solid #e5e5e5; border-radius: 10px; padding: 12px; background: #fff; }
.actions { display: flex; gap: 8px; flex-wrap: wrap; margin: 8px 0; }
.logs { max-height: 200px; overflow: auto; border: 1px solid #eee; border-radius: 6px; padding: 8px; background: #fafafa; }
.log-line { font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, 'Liberation Mono', 'Courier New', monospace; font-size: 12px; padding: 2px 0; }
.muted { color: #777; font-size: 12px; }
.row-between { display: flex; align-items: center; justify-content: space-between; gap: 8px; }
.kv { display: grid; grid-template-columns: 140px 1fr; gap: 8px; align-items: center; margin-bottom: 8px; }
.kv input { width: 100%; padding: 6px 8px; border: 1px solid #ddd; border-radius: 6px; }
</style>