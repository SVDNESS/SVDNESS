The implementation works as follows:
1. NPCs are spawned using client coordinates.
2. GeoService determines the valid ground position.
3. The corrected Z value is applied to the spawn.
4. The updated Z value is written back to the XML file.

VisibleObjectSpawner:
public static VisibleObject spawnNpc(SpawnTemplate spawn, int instanceIndex) {
		int npcId = spawn.getNpcId();
    var npcTemplate = DataManager.NPC_DATA.getNpcTemplate(npcId);
		if (npcTemplate == null) {
            log.error("No template for NPC {}.", npcId);
			return null;
		}
    var npc = new Npc(new NpcController(), spawn, npcTemplate);
		npc.setCreatorId(spawn.getCreatorId());
		npc.setMasterName(spawn.getMasterName());
    npc.setKnownlist(npc.isFlag() ? new FlagKnownList(npc) : new NpcKnownList(npc));
		npc.setEffectController(new EffectController(npc));
		if (WalkerFormator.processClusteredNpc(npc, spawn.getWorldId(), instanceIndex)) {
			return npc;
		}
		try {
			SpawnEngine.bringIntoWorld(npc, spawn, instanceIndex);
		} catch (Exception ex) {
			log.error("Error during spawn of npc {}, world {}, x-y {}-{}.", npcTemplate.getTemplateId(), spawn.getWorldId(), spawn.getX(), spawn.getY());
			log.error("Npc {} will be despawned.", npcTemplate.getTemplateId(), ex);
			npc.getController().delete();
		}
		//Corrects invalid client-side spawn Z coordinates by snapping NPCs to the nearest valid ground surface using GeoService.
		if (GeoDataConfig.NEW_REVENGE_GEO_ENABLE && GeoDataConfig.CORRECTS_NPC_COORDINATES_ENABLE) {
			float currentZ = npc.getZ();
			//Cast from slightly above the current Z (+0.5) to detect the terrain below the NPC.
			//If the NPC is already on the ground, the returned Z will be unchanged and no correction is applied.
			//If the NPC is above the terrain, the ground Z is detected and used to correct the spawn position.
			float geoZ = GeoService.getInstance().getZ(npc.getWorldId(), npc.getX(), npc.getY(), currentZ + 0.5f, 0.0f, npc.getInstanceId());
			if (!Float.isNaN(geoZ) && Math.abs(geoZ - currentZ) > 0.5f) {
				npc.getPosition().setZ(geoZ);
				spawn.setZ(geoZ);
			}
		}
		return npc;
	}

New Class:
/**
* One-time utility that takes GeoService-corrected Z values from SpawnTemplate (adjusted during NPC spawning) and writes them back into the spawn XML file.
* After the first execution, the XML file will contain the correct Z coordinates, so runtime correction will no longer be required on subsequent server starts (diff < 0.5f).
* Must be called from GameServer only after SpawnEngine.spawnAll().
* Example:
* SpawnZXmlPatcher.patch(800030000, "./data/static_data/spawns/Npcs/SVDNESS/800030000_Rafslan2.xml");
*/
public class SpawnZXmlPatcher {
    private static final Logger log = LoggerFactory.getLogger(SpawnZXmlPatcher.class);
    private static final float MIN_DIFF = 0.5f;
    private static final Pattern X_PATTERN = Pattern.compile("x=\"([^\"]+)\"");
    private static final Pattern Y_PATTERN = Pattern.compile("y=\"([^\"]+)\"");
    private static final Pattern Z_PATTERN = Pattern.compile("z=\"([^\"]+)\"");
    private static final Pattern NPC_ID_PATTERN = Pattern.compile("npc_id=\"(\\d+)\"");

    private SpawnZXmlPatcher() {}

    public static void patch(int worldId, String xmlFilePath) {
        File xmlFile = new File(xmlFilePath);
        if (!xmlFile.isFile()) {
            log.warn("SpawnZXmlPatcher: file not found: {}.", xmlFilePath);
            return;
        }
        List<SpawnGroup> groups = DataManager.SPAWNS_DATA.getSpawnsByWorldId(worldId);
        if (groups.isEmpty()) {
            log.warn("SpawnZXmlPatcher: no spawn groups found for worldId {}.", worldId);
            return;
        }
        Map<Integer, List<SpawnTemplate>> byNpcId = new HashMap<>();
        for (var group : groups) {
            byNpcId.computeIfAbsent(group.getNpcId(), _ -> new ArrayList<>()).addAll(group.getSpawnTemplates());
        }
        try {
            String content = Files.readString(xmlFile.toPath(), StandardCharsets.UTF_8);
            String[] lines = content.split("\n", -1);
            int corrected = 0;
            int currentNpcId = -1;
            for (int i = 0; i < lines.length; i++) {
                String line = lines[i];
                //Отслеживаем текущий npc_id.
                Matcher npcMatcher = NPC_ID_PATTERN.matcher(line);
                if (npcMatcher.find()) {
                    currentNpcId = Integer.parseInt(npcMatcher.group(1));
                }
                //Обрабатываем строки со спотами.
                if (!line.contains("<spot ") || currentNpcId == -1) {
                    continue;
                }
                List<SpawnTemplate> templates = byNpcId.get(currentNpcId);
                if (templates == null) {
                    continue;
                }
                Matcher xm = X_PATTERN.matcher(line);
                Matcher ym = Y_PATTERN.matcher(line);
                Matcher zm = Z_PATTERN.matcher(line);
                if (!xm.find() || !ym.find() || !zm.find()) {
                    continue;
                }
                float x = Float.parseFloat(xm.group(1));
                float y = Float.parseFloat(ym.group(1));
                float xmlZ = Float.parseFloat(zm.group(1));
                float correctedZ = findZ(templates, x, y);
                if (!Float.isNaN(correctedZ) && Math.abs(correctedZ - xmlZ) > MIN_DIFF) {
                    //Заменяем только значение z — порядок атрибутов сохраняется.
                    lines[i] = line.replace("z=\"" + zm.group(1) + "\"", "z=\"" + correctedZ + "\"");
                    corrected++;
                }
            }
            if (corrected == 0) {
                log.info("SpawnZXmlPatcher: no corrections needed for {}.", xmlFile.getName());
                return;
            }
            File backup = new File(xmlFilePath + ".bak");
            Files.copy(xmlFile.toPath(), backup.toPath(), StandardCopyOption.REPLACE_EXISTING);
            Files.writeString(xmlFile.toPath(), String.join("\n", lines), StandardCharsets.UTF_8);
            log.info("SpawnZXmlPatcher: patched {} Z values in {}. Backup: {}.", corrected, xmlFile.getName(), backup.getName());
        } catch (Exception e) {
            log.error("SpawnZXmlPatcher: failed to patch {}.", xmlFilePath, e);
        }
    }

    private static float findZ(List<SpawnTemplate> templates, float x, float y) {
        for (var t : templates) {
            if (t.getX() == x && t.getY() == y) {
                return t.getZ();
            }
        }
        return Float.NaN;
    }
}

And called from GameServer after SpawnEngine.spawnAll():
SpawnEngine.spawnAll();
if (GeoDataConfig.NEW_REVENGE_GEO_ENABLE && GeoDataConfig.CORRECTS_NPC_COORDINATES_ENABLE) {
    SpawnZXmlPatcher.patch(800030000, "./data/static_data/spawns/Npcs/SVDNESS/800030000_Rafslan.xml");
}
