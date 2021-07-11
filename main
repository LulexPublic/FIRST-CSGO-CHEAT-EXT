#include <Windows.h>
#include <TlHelp32.h>
#include <iostream>
#include <chrono>
#include <thread>

int triggerbot_key = VK_MENU; // menu key using Virtual Keys

#pragma region variables, offsets, etc..  using pragma to define specific usage area

struct vars // declaring configs by default
{
	struct triggerbot
	{
		bool enabled = true;
		bool team_check = false;
	}triggerbot;

	struct rcs
	{
		bool enabled = true;
	}rcs;

	struct glow
	{
		bool enabled = true;
	}glow;

	struct misc
	{
		bool bunnyhop = true;
		bool radar = true;
	}misc;
}vars;

struct off // declaring offsets
{
	uintptr_t local_player = 0xD2EB94;
	uintptr_t entity_list = 0x4D42A34;
	uintptr_t force_attack = 0x3173FF0;
	uintptr_t force_jump = 0x51EC6B0;
	uintptr_t crosshair_id = 0xB3D4;

	uintptr_t shots_fired = 0xA380;
	uintptr_t fl_sensor_time = 0x3964;
	uintptr_t b_spotted = 0x93D;
	uintptr_t flags = 0x104;
	uintptr_t health = 0x100;
	uintptr_t team = 0xF4;
	uintptr_t dormant = 0xED;

	uintptr_t aim_punch_angle = 0x302C;
	uintptr_t client_state = 0x589DCC;
	uintptr_t client_state_view_angles = 0x4D88;
}off;

struct vec3 // simple vector function unstable af xD
{
	float x, y, z;
};

HANDLE handle; // calling a handle
DWORD process_id;// calling processid

DWORD client; // caling client.dll
DWORD engine;// calling engine funtion

#pragma endregion

#pragma region attach to process & getting module functions

bool attach(const char* process_name) // stolen attaching method works for all thread processess
{
	PROCESSENTRY32 proc_entry32;
	proc_entry32.dwSize = sizeof(PROCESSENTRY32);

	HANDLE handle_process_id = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0);

	while (Process32Next(handle_process_id, &proc_entry32))
	{
		if (!strcmp(proc_entry32.szExeFile, process_name))
		{
			process_id = proc_entry32.th32ProcessID;
			handle = OpenProcess(PROCESS_ALL_ACCESS, 0, process_id);
			CloseHandle(handle_process_id);
			return true;
		}
	}
	CloseHandle(handle_process_id);
	return false;
}

DWORD get_module(const char* module_name) // getting module
{
	MODULEENTRY32 mod_entry32;
	mod_entry32.dwSize = sizeof(MODULEENTRY32);

	HANDLE handle_module = CreateToolhelp32Snapshot(TH32CS_SNAPMODULE, process_id);

	while (Module32Next(handle_module, &mod_entry32))
	{
		if (!strcmp(mod_entry32.szModule, module_name))
		{
			CloseHandle(handle_module);
			return (DWORD)mod_entry32.modBaseAddr;
		}
	}
	CloseHandle(handle_module);
	return false;
}

#pragma endregion

#pragma region write/read process memory functions

template <class T>
void write(DWORD address, T data) // write process memory function 
{
	WriteProcessMemory(handle, (LPVOID)address, &data, sizeof(T), 0);
}

template <class T>
T read(DWORD address) // read process memory function
{
	T data;
	ReadProcessMemory(handle, (LPVOID)address, &data, sizeof(T), 0);
	return data;
}

#pragma endregion

#pragma region game functions

DWORD get_local_player() // read local player addres
{
	return read<DWORD>(client + off.local_player);
}

DWORD get_entity_from_index(int i) // read entity index, kinda useless its 1-9 for T players 10-19 for CT 
{
	return read<DWORD>(client + off.entity_list + i * 0x10);
}

int get_health(DWORD entity) // read health addres from specific entity
{
	return read<int>(entity + off.health);
}

int get_team(DWORD entity) // read is ally or enemy from entity kinda useless it can be hardcoded as  return 1<<9<<19 
{
	return read<int>(entity + off.team);
}

bool is_dormant(DWORD entity) // read dormant from entity we need this for checking if we read entities properly
{
	return read<bool>(entity + off.dormant);
}

#pragma endregion

#pragma region features

void triggerbot() // trigger func
{
	if (!vars.triggerbot.enabled) // disabling when off in config
		return;

	DWORD local_player = get_local_player(); // defining local player 
	int crosshair = read<int>(local_player + off.crosshair_id); // reading crosshair positon
	DWORD entity = read<DWORD>(client + off.entity_list + (crosshair - 1) * 0x10); // checking for specific entity in crosshair range
	int entity_team = get_team(entity); // checking team
	int local_team = get_team(local_player); // checking player
	int entity_health = get_health(entity); // checking health
	bool dormant = is_dormant(entity); // checking dormancy

	if (crosshair != 0 && entity_health != 0 && !dormant && GetAsyncKeyState(triggerbot_key) & 0x8000) // checking if no crosshair is visible ( knife, blabla no grenade check tho xD), checking dormancy and if key is held
	{
		if (local_team == entity_team && vars.triggerbot.team_check) // another obvious checks
			return;

		write<int>(client + off.force_attack, 6); // we write our on function into the client, then server recieves its data. THIS FUNCTION IS DETECTED AF do a trampoline trick to avoid it example:
		// read all data -> save all data -> write new data -> write old data  do it before the server reads from the client no problem ^^
	}
}

vec3 old_angles; // checking direction where do we aim
void rcs()
{
	if (!vars.rcs.enabled)
		return;

	DWORD local_player = get_local_player();
	int shots_fired = read<int>(local_player + off.shots_fired);
	vec3 aim_punch = read<vec3>(local_player + off.aim_punch_angle);

	vec3 rcs_angles;

	if (local_player && shots_fired > 1) // do not move an angle while shooting a headshot XD
	{
		DWORD client_state = read<DWORD>(engine + off.client_state);
		vec3 view_angles = read<vec3>(client_state + off.client_state_view_angles);

		rcs_angles.x = (view_angles.x + old_angles.x) - aim_punch.x * 2; // correcting angles
		rcs_angles.y = (view_angles.y + old_angles.y) - aim_punch.y * 2;

		write<vec3>(client_state + off.client_state_view_angles, rcs_angles);

		old_angles.x = aim_punch.x * 2;
		old_angles.y = aim_punch.y * 2;
	}
	else
	{
		old_angles.x = 0; // reseting old angles
		old_angles.y = 0;
	}
}

void glow()
{
	if (!vars.glow.enabled)
		return;

	for (int i = 1; i <= 64; ++i) // scanning for all entites on map <=19 crashes the get_entity_from_index function and we need it so it has to be 64 :[ here optimization is dead, you can hard code enemy entities and write memory only to these values
	{
		DWORD entity = get_entity_from_index(i);

		if (entity)
			write<float>(entity + off.fl_sensor_time, 86400); // writing memory detected af, sensortime is useless HARDCODED value by valve idk why
	}
}

void bhop()
{
	if (!vars.misc.bunnyhop)
		return;

	DWORD local_player = get_local_player();
	int flags = read<int>(local_player + off.flags);

	if (local_player && flags == 257 && GetAsyncKeyState(VK_SPACE) & 0x8000) // checking flag mainly and if space is held 0x8000 is negative DORMANT but we cant dynamicly do something like dormant!=dormant :p volvo
		write<int>(client + off.force_jump, 6);
}

void radar()
{
	if (!vars.misc.radar)
		return;

	for (int i = 1; i <= 64; ++i)
	{
		DWORD entity = get_entity_from_index(i);

		if (entity)
			write<bool>(entity + off.b_spotted, true); // writing detected af memory, trampoline for radar doesnt work, you can abuse it by not calling spotted as true maybe no vac -.-
		// example:   read player -> checking if is spotted -> if no write memory with old data -> calling the function true -> read the data again if  old!=new -> write with new data ^^
	}
}

#pragma endregion

int main()
{
	if (attach("csgo.exe")) // attaching csgo
	{
		client = get_module("client_panorama.dll"); // it was not panorama when I made it :D
		engine = get_module("engine.dll");

		while (true) //calling all functions 24/7
		{
			triggerbot();
			rcs();
			glow();
			bhop();
			radar();

			std::this_thread::sleep_for(std::chrono::milliseconds(1)); // standalone loop hack not to burn PC its unoptimized af there would be better making a mark to a point before a loop, something like a bed in minecraft ^^
		}
	}
	exit(0);
}
